# base44-migrate

Playbook and AI agent skill for migrating a **Base44** app (`@base44/sdk`, `@base44/vite-plugin`, Deno functions, JSONC entities) to a self-hosted **Node.js + Express + PostgreSQL/MongoDB + Docker** stack—one module at a time, without a big-bang rewrite.

---

## Files

### [`SKILL.md`](./SKILL.md) — The Migration Playbook
An end-to-end, 6-phase execution guide for an AI coding agent (or developer) to follow in order:
- **Phase 0 — Audit & Replacement Interview:** Scan `package.json`, `base44/`, and `src/` and confirm stack replacements (DB, auth, email, storage, cron, hosting) before writing code.
- **Phase 1 — Facade & Adapter Contract:** Keep UI importing `{ base44 }` from `@/api/base44Client`, build `localAdapter.js`, feature-switch via `VITE_LOCAL_MODULES`, and enforce parity with `adapterContract.test.js`.
- **Phase 2 — Generic Backend Core:** Self-host `.jsonc` schemas with one generic repository, query compiler, RLS evaluator, and JWT + rotating `HttpOnly` refresh auth.
- **Phase 3 — Vertical-Slice Porting:** Port one module at a time using the mechanical Deno-to-Express translation table.
- **Phase 4 — Guardrails:** Catch real-world traps (array vs `.items` parity, heavy client `.list()` queries, `no-undef`/TDZ blank screens, `ErrorBoundary` + `ErrorLog`).
- **Phase 5 — Cutover & Verification:** Strip `@base44/sdk` and `@base44/vite-plugin`, archive `base44/`, and pass lint, tests, build, route smoke, and `/api/health`.

### [`verification.md`](./verification.md) — The Verification Rubric
A cold-start test protocol and pass/fail gate checklist to verify a migration (or benchmark how reliably an AI agent executes `SKILL.md` on a fresh repo):
- **Cold-Start Dry Run:** Run a fresh agent session with only `SKILL.md` and score every phase.
- **Hard Gates (A–D):** Concrete checkboxes covering audit discipline, facade preservation, generic CRUD/RLS, auth security, ESLint blank-page guards, and final `make lint` / `make test` / `make build` / Playwright smoke checks.
- **A/B Benchmark:** Optional comparison of human corrections and time-to-green with vs. without `SKILL.md`.

---

## How to Use

### Option 1 — As an AI Agent Skill (Recommended)
Clone into your target Base44 project (or your coding agent's skills directory):

```bash
git clone https://github.com/X0EF/base44-migrate.git
```

Then start a session in your Base44 repo and prompt:

> Remove Base44 from this project using `base44-migrate/SKILL.md`.

Once the agent finishes, grade the result against [`verification.md`](./verification.md).

### Option 2 — As a Manual Playbook
Follow [`SKILL.md`](./SKILL.md) phase by phase on a feature branch, using `VITE_LOCAL_MODULES` to cut over one module at a time while keeping the rest of the app running, then run the Gate D checks in [`verification.md`](./verification.md) before removing `base44/`.
