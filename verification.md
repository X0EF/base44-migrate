# Base44 Migrate Verification Rubric (`verification.md`)

Use this checklist to test how capable `base44-migrate/SKILL.md` is on a fresh repository with a cold-start agent (zero prior conversation history).

---

## 1. Cold-Start Dry Run Protocol
1. Open a fresh agent session on a Base44 repository (or a clean branch of one) with **only** `base44-migrate/SKILL.md` provided.
2. Prompt: *"Remove Base44 from this project using `base44-migrate/SKILL.md`."*
3. Score every gate below (`PASS` / `FAIL`).

---

## 2. Hard Gates

### Gate A — Phase 0 (Audit & Interview)
- [ ] Agent scans `package.json`, `base44/`, and `src/` **before** editing files.
- [ ] Agent lists every Base44 feature found: Auth, Entities/RLS, Functions, Shared Libs, Automations/Cron, Integrations, Build Plugins.
- [ ] Agent **stops and asks** the user to confirm replacement choices (DB, Auth, Email, Storage, Hosting) before writing code.

### Gate B — Phase 1 & 2 (Facade & Generic Core)
- [ ] UI still imports `{ base44 }` from `@/api/base44Client` (no raw `fetch()` rewrites across pages).
- [ ] `adapterContract.test.js` exists and verifies every `base44.*` call in `src/` exists on `localAdapter.js`.
- [ ] Zero custom per-entity CRUD route files — handled by generic `entityRepository.js` + `queryCompiler.js` + JSONC RLS.
- [ ] Auth uses hashed passwords (`bcrypt`), JWT + rotating `HttpOnly` refresh token, and locks `role`/`department` on self-update.

### Gate C — Phase 3 & 4 (Vertical Slices & Traps)
- [ ] Functions ported mechanically (`(base44, body = {})` → `{ status, data }`), keeping `shared/` intact.
- [ ] Array vs `.items` parity enforced (`asList()` helper + zero raw `.(list|filter)(...).items`).
- [ ] ESLint enforces `no-undef`, `no-use-before-define`, and `react-hooks/rules-of-hooks`.
- [ ] React `ErrorBoundary` wraps `<Outlet />` + unhandled crashes / `500`s logged to DB (`ErrorLog`).

### Gate D — Phase 5 (Cutover & Automated Checks)
- [ ] Zero `@base44/sdk` or `@base44/vite-plugin` references left in `package.json`, `vite.config.*`, or `src/`.
- [ ] `make lint` exits `0`.
- [ ] `make test` exits `0`.
- [ ] `make build` exits `0`.
- [ ] Playwright route smoke (`smoke-routes.mjs`) renders all primary routes with `0` `pageerror`s and `0` blank screens.
- [ ] `/api/health` returns `{"status":"ok","db":"connected"}`.

---

## 3. A/B Benchmark (Optional)
Run the same 2-module Base44 sample repo **with** `SKILL.md` vs **without** `SKILL.md` and compare:
1. Total human corrections needed (target: **≤ 1** with skill).
2. Time to `make lint && make test && make build` green.
3. Any trap the agent still hit → add directly to **Phase 4** of `SKILL.md`.
