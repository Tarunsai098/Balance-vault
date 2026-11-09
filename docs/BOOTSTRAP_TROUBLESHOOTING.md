# First-admin bootstrap troubleshooting

Use this when `npm run admin:bootstrap -w apps/api` fails. The CLI prints the stage (`connect`, `set_role`, `bootstrap_function`, ...) and the SQLSTATE code. Never grant broader privileges until the checks below show which layer is failing.

## Quick diagnosis by SQLSTATE

| Code | Meaning | Check |
| --- | --- | --- |
| `28P01` / `28000` | Bad credentials | Bootstrap login password and URL |
| `42501` at `set_role` | Login cannot `SET ROLE` | Membership check (1) |
| `42501` at `bootstrap_function` | Operator role cannot execute the function | Function ACL check (2) |
| `55000` | An admin already exists | Bootstrap is one-time; log in instead |
| `22023` | Input rejected by the function | Name/email/password hash format |

## Read-only verification queries

Run as the Neon project owner against the direct endpoint. They change nothing.

```sql
-- (1) Login exists, can log in, and is a member of the operator role.
--     NOINHERIT is expected: membership permits SET ROLE but grants nothing implicitly.
SELECT r.rolname, r.rolcanlogin, r.rolinherit,
       pg_has_role(r.rolname, 'balancevault_bootstrap_operator', 'MEMBER') AS is_member,
       pg_has_role(r.rolname, 'balancevault_bootstrap_operator', 'USAGE')  AS inherits_privileges,
       pg_has_role(r.rolname, 'balancevault_bootstrap_operator', 'SET')    AS can_set_role
FROM pg_roles r WHERE r.rolname = 'balancevault_bootstrap_login';

-- (2) The operator role may execute the bootstrap function.
SELECT has_function_privilege('balancevault_bootstrap_operator',
       'public.bootstrap_first_admin(uuid,text,text,text)', 'EXECUTE') AS can_execute;

-- (3) Bootstrap state: no rows means not yet bootstrapped.
SELECT * FROM public."PrivilegedBootstrap";

-- (4) Migrations applied.
SELECT migration_name, finished_at FROM _prisma_migrations ORDER BY started_at;
```

## Lessons learned

- Role **membership** is not role **administration**. Being able to `SET ROLE` into a role does not let you `ALTER ROLE ... PASSWORD` on it; on Neon, even the project-owner connection may be refused (SQLSTATE `42501`). Set passwords for login roles with `\password <login>` in `psql`.
- Use a **direct** (non-pooled) Neon host for bootstrap, migrations, and key provisioning. The CLI now wraps `SET LOCAL ROLE` and the function call in one transaction so the role switch cannot be lost on a pooled connection.
- Do not substitute the project owner for the bootstrap login to "make it work"; that bypasses the isolation the design depends on.
