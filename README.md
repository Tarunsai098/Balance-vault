# BalanceVault — money movement with a ledger you can prove

A secure account management platform with role-based access, user onboarding, and auditable account operations.

> **Hackathon demonstration.** Uses synthetic data only. It is not a production financial service and must not handle real funds.

## The problem

Managing financial accounts and their operations requires more than storing account data. It requires controlled access, secure authentication, clear onboarding, and accountability for important actions. BalanceVault brings these together around structured account management, role-based access, onboarding workflows, and auditability.

## What I built

- React (Vite) frontend, deployed on Vercel.
- Node.js/Express backend with API routes.
- PostgreSQL via Prisma, with migrations and separate database roles for different responsibilities (runtime, auth read-only, migration owner, function owners, bootstrap operator).
- Authentication and role-based access control (`ACCOUNT_HOLDER`, `ADMIN`, `AUDITOR`).
- Account-holder registration and account-opening/onboarding workflows with administrator approval.
- Admin functionality for user management, one-time privileged invitations, onboarding review, and audit-log access.
- Double-entry ledger: integer paise, append-only entries, idempotent posting performed by protected PostgreSQL functions.
- Redis-backed rate limiting on the backend.
- Backend health (`/api/health`) and readiness (`/api/ready`) endpoints.

## Current limitation (honest status)

The deployment infrastructure and health/readiness checks are verified. The **first-administrator bootstrap and the complete end-to-end admin login flow are still being tested** on the live deployment. I do not claim those features are verified there until they work. See [bootstrap troubleshooting](docs/BOOTSTRAP_TROUBLESHOOTING.md).

What is verified by automated tests (per [development handoff](docs/DEVELOPMENT_HANDOFF.md)): backend and PostgreSQL integration suite and frontend suite against disposable databases. What is **not** yet verified on the live deployment:

- First-admin bootstrap and admin login against the deployed backend.
- End-to-end registration and onboarding in the browser.
- Admin user management and audit-log views through the live interface.
- Browser-based testing of the deployed app; deployment monitoring and operational reliability.

## How to run it

Requirements: Node.js 24, npm 10/11, PostgreSQL 17, Redis. Full local setup is under [Local development](#local-development).

```sh
npm ci
cp .env.example .env        # configure local DATABASE_URL, AUTH_DATABASE_URL, REDIS_URL, secrets
npm run db:generate
npm run db:migrate          # needs MIGRATION_DATABASE_URL pointing at a dev database
npm run dev                 # frontend :5173, API :3001
```

For a local demonstration, both the frontend and backend must use the local database and Redis instances.

## AI tools I used

- **ChatGPT** — technical explanations, architecture discussions, debugging, deployment troubleshooting.
- **OpenAI Codex in VS Code** — code implementation, repository inspection, code review.

Representative prompts are in [docs/PROMPTS.md](docs/PROMPTS.md).

## What the AI got wrong, and what I fixed myself

During deployment, the AI confused PostgreSQL **role membership** with permission to **administer** another role. It suggested changing a role password through the project-owner connection; Neon rejected it with SQLSTATE `42501`.

Rather than granting broader privileges, I inspected the actual configuration and found that the bootstrap login existed with `LOGIN`, was a member of the bootstrap operator role, that membership allowed `SET ROLE`, and that the operator could execute the admin bootstrap function. So the original permission assumptions were incomplete, and the failure was elsewhere. Reviewing the bootstrap path also led me to make the CLI run `SET LOCAL ROLE` and the function call in a single transaction, so it is safe even if a connection is pooled. The lesson: AI-generated suggestions must be checked against the real database configuration and execution environment.

## What is not finished

- Complete and verify the first-admin bootstrap on the live deployment.
- Verify admin login against the deployed backend.
- Test registration and onboarding end to end.
- Verify admin user management and audit-log functionality through the live interface.
- Complete browser-based testing of the deployed application.
- Improve deployment monitoring and operational reliability.

---

## Local development

Requirements: Node.js 24, npm 10 or 11, PostgreSQL 17, and a Redis-compatible service for auth rate limiting.

```sh
npm ci
cp .env.example .env
# Set DATABASE_URL to a local login that inherits balancevault_runtime.
# Set JWT_ACCESS_SECRET with: openssl rand -base64 48
# Start local Redis and set REDIS_URL=redis://127.0.0.1:6379/0
npm run dev
```

The frontend runs at `http://localhost:5173` and the API at `http://localhost:3001`. Run either app with `npm run dev:web` or `npm run dev:api`.

## Database roles and migrations

Provision roles with an administrator by reviewing and applying `apps/api/prisma/roles.sql`. Create the database owned by `balancevault_migration_owner`; run Prisma migrations only with `MIGRATION_DATABASE_URL`. The API uses `DATABASE_URL` for ordinary queries and a separate read-only `AUTH_DATABASE_URL` for password verification. The latter alone can read `User.passwordHash`; neither login may inherit migration or function-owner roles. Never set migration credentials on the API service.

```sh
npm run db:generate
npm run db:migrate
```

### First administrator and privileged invitations

Public registration always creates an `ACCOUNT_HOLDER`. After applying the privileged-provisioning migration and provisioning `balancevault_bootstrap_operator`, an authorized operator runs `npm run admin:bootstrap -w apps/api` with a separate `BALANCEVAULT_BOOTSTRAP_DATABASE_URL`. The CLI confirms the database name, prompts for the first administrator's details, hides password input, and sends only a bcrypt hash to the one-time PostgreSQL bootstrap function. Do not configure this connection on the API service.

An authorized database administrator creates the operator login separately, after applying `apps/api/prisma/roles.sql` and the migrations:

```sql
CREATE ROLE balancevault_bootstrap_login LOGIN NOINHERIT;
GRANT balancevault_bootstrap_operator TO balancevault_bootstrap_login;
```

Set its password interactively with `\password balancevault_bootstrap_login` in `psql`, then use that login only in `BALANCEVAULT_BOOTSTRAP_DATABASE_URL`. Do not put the password in this repository or the API service environment.

After bootstrap, an administrator can create one-time ADMIN or AUDITOR invitations in User Management. Copy and securely deliver the invitation link; it is shown once and expires after 48 hours. The invitee chooses their own password. Invitation tokens are 256-bit random values; only SHA-256 hashes are stored. Invitations can be revoked before acceptance. Email delivery is not configured in this project.

The initial migration explicitly switches to `balancevault_posting_owner` while creating its private functions and setting their ACLs. This fixes the migration role's `NOINHERIT` ownership boundary: membership permits an explicit `SET ROLE`, but does not otherwise confer owner privileges. The SQL change is limited to migration privilege sequencing; it does not change financial function bodies or their runtime grants. Because the already-published initial migration checksum changes, databases that recorded the previous version must not be redeployed against it without a deliberate Prisma migration-history reconciliation. New environments should use the corrected history from the outset.

## Deployment

For the fastest free hackathon demo, follow the [Vercel + Render + Neon + Upstash deployment guide](docs/DEMO_DEPLOYMENT.md). The API starts with `npm start`; Render's blueprint generates the Prisma client during its build. Vercel builds only the frontend with `npm run build -w apps/web`. For broader deployment and recovery guidance, see [Deployment configuration](docs/DEPLOYMENT.md).

Useful commands:

```sh
npm test
npm run build
npm run db:migrate
```

See [API documentation](docs/API.md) and [architecture](docs/ARCHITECTURE.md).

See [frontend authentication and session storage](docs/FRONTEND.md) for the browser token strategy and its security limitations.

## Implemented workflows

- Account-holder registration, login, profile updates, password changes, and logout.
- Account opening requests with administrator approval or rejection.
- Account balances, deposits, withdrawals, transfers, transaction history, and paginated statements.
- Administrator account and user management, one-time privileged invitations, and audit views.
- Read-only auditor views restricted to assigned customer accounts.

Financial posting uses a PostgreSQL-controlled double-entry ledger. Amounts are integer paise, ledger entries are append-only, account locks prevent concurrent overdrafts, and idempotency keys make retries safe. The frontend does not update balances or decide authorization.

### Read-only ledger reconciliation

Run `npm run ledger:reconcile -w apps/api` with `RECONCILIATION_DATABASE_URL` set to a least-privilege login that inherits only `balancevault_runtime`. The command checks that every stored account balance equals the sum of its signed ledger entries inside one repeatable-read, read-only transaction. It prints mismatches and exits with status 2 if any are found; it never repairs balances. Run it after data migrations/imports and on an operator-approved schedule, and investigate discrepancies before taking corrective action.

## Architecture and security

The API is organized into Express routes, services, database clients, and middleware under `apps/api/src`. PostgreSQL owns posting, balances, audit events, onboarding, and privileged provisioning logic. `DATABASE_URL` is for ordinary runtime operations; `AUTH_DATABASE_URL` is a separate read-only identity for password-hash verification. Redis backs shared authentication rate limits. See [architecture](docs/ARCHITECTURE.md), [API reference](docs/API.md), and [threat model](docs/THREATS.md).

Environment variable names and their purpose are listed in `.env.example` and [deployment configuration](docs/DEPLOYMENT.md). Never put secret values in source control. Production requires HTTPS, exact HTTPS `WEB_ORIGIN` entries, PostgreSQL TLS, a dedicated Redis service, and separate migration and runtime credentials.

## Testing and release checks

```sh
npm test
npm test -w apps/web
npm run db:generate
npx prisma validate --config apps/api/prisma.config.js
npm run build
git diff --check
```

PostgreSQL integration tests run only when the `BALANCEVAULT_TEST_*` variables point at disposable databases ending in `_test`. Their setup and role requirements are in [development handoff](docs/DEVELOPMENT_HANDOFF.md). Do not point these variables at `balancevault_dev` or production.

## Deployment and recovery

Use the controlled checklist in [deployment configuration](docs/DEPLOYMENT.md). Back up before schema releases with `pg_dump --format=custom --file=balancevault-$(date +%Y%m%dT%H%M%S).dump "$DATABASE_URL"`; restore into a separate recovery database with `pg_restore --no-owner --dbname="$RECOVERY_DATABASE_URL" backup.dump`, then validate before any cutover. Keep the backup encrypted and access-controlled. Do not restore over an existing database as a routine rollback; use a reviewed recovery plan.

AI-assisted development history is recorded only when supported by an available prompt or session record in [PROMPTS.md](docs/PROMPTS.md). Current implementation status and known limits are in [development handoff](docs/DEVELOPMENT_HANDOFF.md).
