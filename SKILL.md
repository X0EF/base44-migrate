---
name: base44-migrate
description: End-to-end playbook for migrating a Base44 application to a self-hosted Node.js + Express + PostgreSQL (or MongoDB) + Docker stack without a big-bang rewrite.
---

# Base44 Migrate (`SKILL.md`)

A complete, one-stop migration playbook for removing `@base44/sdk` and `@base44/vite-plugin` from a React + Vite codebase and replacing them with a self-hosted backend.

---

## Phase 0 — Audit & Replacement Interview *(Do This First)*

Before writing any code, scan the repository (`package.json`, `base44/`, `src/`) and list **every Base44 capability** the project actually uses:

1. **Auth & Users:** `base44.auth.me`, `login`, `logout`, `invite`, `accept-invite`, `reset-password`, custom user fields (`role`, `department`, `additional_departments`).
2. **Entities & RLS:** All `base44/entities/*.jsonc` schemas and every `base44.entities.<Name>.<op>` call in `src/` (`.list`, `.filter`, `.get`, `.create`, `.update`, `.delete`, `.count`, `.aggregate`, `.bulkUpdate`, `.subscribe`).
3. **Server Functions & Shared Libs:** All `base44/functions/*/entry.ts` (Deno) and `base44/shared/*.ts`.
4. **Automations / Cron:** Scheduled jobs and entity triggers (`create` / `update` hooks).
5. **Integrations & Plugins:** `UploadFile`, `SendEmail`, `InvokeLLM`, `@base44/vite-plugin`, `@base44/sdk`.

### Stop & Confirm Replacements With the User
Present the inventory above and ask the user to confirm what replaces each piece:
- **Database:** PostgreSQL (JSONB + typed indexes, recommended) or MongoDB?
- **Auth:** Local JWT + rotating `HttpOnly` refresh cookie + `bcrypt`?
- **Email:** Nodemailer (SMTP) / Resend / dev console logger?
- **File Uploads:** Local `/uploads` disk, S3/R2, or Cloudinary?
- **Cron Jobs:** `node-cron` + `/api/jobs/:name` (`X-Cron-Secret`)?
- **Hosting & Runtime:** Docker Compose locally + Render / Fly / VPS in prod?

Do **not** skip this step.

---

## Phase 1 — Facade Pattern & Adapter Contract *(Zero Big-Bang Rewrite)*

Never rewrite UI pages to call `fetch()` directly. Keep all UI code importing `{ base44 }` from `@/api/base44Client`.

1. **Create `src/api/localAdapter.js`:**
   - Implement the exact SDK surface used by `src/`:
     - `auth`: `me()`, `login()`, `logout()`, `refresh()`, `updateMe()`
     - `entities.<Entity>`: `list(sort, limit)`, `filter(query, optionsOrSort, limit)`, `get(id)`, `create(data)`, `update(id, patch)`, `delete(id)`, `bulkUpdate(items)`, `count(query)`, `aggregate(spec)`, `subscribe(cb)`
     - `functions.invoke(fnName, body)` → `POST /api/functions/:fnName`
     - `integrations.Core.UploadFile` / `SendEmail` → `/api/integrations/*`
2. **Feature-Switch in `src/api/base44Client.js`:**
   - Route modules via `VITE_LOCAL_MODULES` (one module at a time, or `all`) so migrated slices run locally while unmigrated ones keep working.
3. **Add `adapterContract.test.js`:**
   - Automatically scan `src/` for every `base44.entities.*`, `base44.auth.*`, `base44.integrations.Core.*` call and assert the method exists on `localAdapter.js`.
   - Assert **zero** raw `base44.entities.*.(list|filter)(...).items` calls (return plain arrays consistently).

---

## Phase 2 — Generic Backend Core

Run everything inside Docker (`Makefile`: `make up`, `make seed`, `make test`, `make lint`, `make build`) — never install DBs or Node versions directly on the host OS.

1. **Self-Host Entity Schemas:**
   - Copy `base44/entities/*.jsonc` into `server/src/schemas/entities/*.jsonc`.
   - Never write custom CRUD routes per entity.
2. **Build One Generic Repository + Query Compiler + RLS Evaluator:**
   - `queryCompiler.js`: compile Mongo-style operators (`$ne`, `$in`, `$nin`, `$gt/$gte/$lt/$lte`, `$regex`, `$exists`, `$or`, `$and`), `sort`, field projection (`fields`), and a safe **default `500`-row cap** (reject unsafe field names to block SQL injection).
   - `entityRepository.js`: load `.jsonc` schemas, enforce JSONC `rls` (`create`, `read`, `update`, `delete` + `asServiceRole` bypass), and expose `registerEntityHook('create' | 'update', EntityName, fn)`.
3. **Auth & Session Security:**
   - JWT access token (`6h`–`24h`) + **30-day rotating `HttpOnly` refresh cookie** (reuse detection revokes the token family).
   - `bcrypt` password hashing; invite + `accept-invite` flow; `must_change_password` flag for non-`super_admin` seeded users.
   - Lock `role` and `department` on `PATCH /api/auth/me` so users cannot escalate privileges.

---

## Phase 3 — Vertical-Slice Porting (`0` → `N`)

Migrate **one sidebar module at a time** in this order:
1. **Update `docs/PRD.md` first** (business rules, roles, invariants).
2. **Port `base44/shared/*.ts` → `server/src/shared/*.js`** once and import from `../shared/`.
3. **Port each `base44/functions/<fnName>/entry.ts`** into `server/src/services/<module>Service.js` using this 4-line mechanical translation:

| Base44 Deno (`entry.ts`) | Self-Hosted Express (`<module>Service.js`) |
|---|---|
| `import ... from '../../shared/foo.ts'` | `import ... from '../shared/foo.js'` |
| `export default async function (req)` | `export async function <fnName>(base44, body = {})` |
| `const base44 = createClientFromRequest(req)` | *(Remove — `base44` is 1st arg)* |
| `const body = await req.json()` | *(Remove — `body` is 2nd arg)* |
| `Deno.env.get('VAR')` | `process.env.VAR` |
| `return Response.json(payload, { status: 400 })` | `return { status: 400, data: payload }` |
| `return Response.json(payload)` | `return { status: 200, data: payload }` |

4. **Register handlers** in `server/src/routes/functions.js`.
5. **Port cron jobs** to `node-cron` + `/api/jobs/:job` guarded by `X-Cron-Secret` or `admin` role.
6. **Seed & Test:** Update `seed.js` and add unit + integration tests (`node --test`) for every slice.

---

## Phase 4 — Real-World Traps & Guardrails

1. **Array vs `.items` Parity:**
   - Legacy Base44 hooks sometimes return `{ items: [...] }` while `localAdapter` returns `[...]`. Use an `asList(res)` helper (`Array.isArray(res) ? res : res?.items || []`) everywhere.
2. **Audit Frontend-Heavy `.list()` Queries:**
   - Replace multi-table client joins and heavy rollups with single backend endpoints (e.g., `globalSearch`, `traceLookup`, `dashboardSummary`) and date-scope large ledger queries.
3. **Prevent Silent Blank Pages:**
   - Vite builds pass even with `ReferenceError` bugs (`no-undef` or TDZ `no-use-before-define`).
   - Enable `no-undef`, `no-use-before-define`, and `react-hooks/rules-of-hooks` in `eslint.config.js` and gate pushes on `make lint`.
   - Wrap `<Outlet />` in a React `ErrorBoundary` and persist client crashes + API `500`s to an `ErrorLog` table in Postgres.
4. **PostgreSQL vs MongoDB Seed Parity:**
   - Wrap Postgres queries in a thin collection shim for `seed.js` rather than calling `db.collection()` directly when `DB_DRIVER=postgres`.

---

## Phase 5 — Cutover & Verification

1. Remove `@base44/sdk` and `@base44/vite-plugin` from `package.json` and `vite.config.js`.
2. Archive or remove `base44/` once all modules are ported and verified.
3. Run the full gate:
   - `make lint` (zero `no-undef`, `no-use-before-define`, `rules-of-hooks`)
   - `make test` (adapter contract + unit + integration tests)
   - `make build`
   - Playwright route smoke test (`scripts/smoke-routes.mjs`) + k6 role-based soak test (`scripts/k6-production-month.js`)
4. Smoke-check `/api/health` (`{"status":"ok","db":"connected"}`) on production after deploy.
