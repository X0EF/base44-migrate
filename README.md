# base44-migrate

An [Agent Skill](https://agentskills.io) and verification rubric for migrating a **Base44** app (`@base44/sdk`, `@base44/vite-plugin`, Deno functions, JSONC entities) to a self-hosted **Node.js + Express + PostgreSQL/MongoDB + Docker** stack—one module at a time, without a big-bang rewrite.

---

## Install

Install directly into your Base44 project using the open skills CLI (works with Claude Code, Cursor, Codex, GitHub Copilot, Gemini/Antigravity, etc.):

```bash
npx skills add X0EF/base44-migrate
```

Install globally across all projects:

```bash
npx skills add X0EF/base44-migrate -g
```

Target a specific agent (e.g. `claude-code`, `cursor`, `codex`):

```bash
npx skills add X0EF/base44-migrate --agent claude-code
```

<details>
<summary>Manual installation (without npx)</summary>

Copy `SKILL.md` and `verification.md` into your agent's skills directory:
- **Standard / Copilot / Codex:** `.agents/skills/base44-migrate/`
- **Claude Code:** `.claude/skills/base44-migrate/`
- **Cursor:** `.cursor/skills/base44-migrate/`

</details>

---

## Usage

Open your AI coding agent inside your Base44 repository and prompt:

> Remove Base44 from this project using the `base44-migrate` skill.

When the migration finishes, grade the output against [`verification.md`](./verification.md).

---

## What's Inside

### [`SKILL.md`](./SKILL.md) — The Migration Skill
The 6-phase playbook the agent follows in order:
1. **Phase 0 — Audit & Replacement Interview:** Scans `package.json`, `base44/`, and `src/`, then confirms stack replacements (DB, auth, email, storage, cron, hosting) with you before editing code.
2. **Phase 1 — Facade & Adapter Contract:** Keeps UI importing `{ base44 }` from `@/api/base44Client`, builds `localAdapter.js`, routes modules via `VITE_LOCAL_MODULES`, and enforces parity with `adapterContract.test.js`.
3. **Phase 2 — Generic Backend Core:** Self-hosts `.jsonc` schemas with one generic repository, query compiler, RLS evaluator, and JWT + rotating `HttpOnly` refresh auth.
4. **Phase 3 — Vertical-Slice Porting:** Ports one module at a time using the mechanical Deno-to-Express translation table.
5. **Phase 4 — Guardrails:** Prevents common traps (array vs `.items` parity, heavy client `.list()` queries, `no-undef`/TDZ blank screens, `ErrorBoundary` + `ErrorLog`).
6. **Phase 5 — Cutover & Verification:** Removes `@base44/sdk` and `@base44/vite-plugin`, archives `base44/`, and passes lint, tests, build, route smoke, and `/api/health`.

### [`verification.md`](./verification.md) — The Verification Rubric
A cold-start audit protocol and pass/fail gate checklist (Gates A–D) to verify a migration is complete and safe—covering audit discipline, facade preservation, generic CRUD/RLS, auth security, ESLint blank-page guards, and `make lint` / `make test` / `make build` / Playwright smoke checks.
