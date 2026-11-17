# Core architecture rules

Apply the rules that fit the project type. A landing page needs sections 1, 6 and 8 only.

## 1. One-way dependencies

- UI depends on services. Services depend on the data layer. Nothing depends upward.
- Pure domain code (types, rules, validators) depends on nothing and is importable by all layers.
- Server-only code never reaches the browser. Enforce it with tooling (the framework's server-only setting, a lint rule on the database driver), not by convention alone.
- Navigation goes through the router. The root component and app shell render the router's outlet; they never choose a page with conditions or state.
- Entry points are thin. `main.ts` or `actions.ts` reads config and composes modules. Logic lives in modules that can be tested without starting a server.

## 2. A feature, end to end

Each feature has the same parts, in this order of dependency:

1. **Domain**: types and pure operations.
2. **Schema**: database tables and a migration.
3. **Service**: business logic. Takes a database handle, knows nothing about HTTP, throws typed errors.
4. **API module**: controller plus `model.ts` (request and response schemas). Maps service errors to status codes.
5. **Loader or client call**: server-side read for the page, typed client for writes.
6. **UI**: page component and feature components.
7. **Tests**: pure logic first, then the service against a real database.

Reads and writes through one writer: when state can be changed from several places (UI, API, background job, AI agent), route all of them through the same operation functions.

## 3. Errors

- Services throw typed error classes (`NotFound`, `Conflict`, `Invalid`).
- Controllers catch the known ones and return a status with a stable code: `{ code: 'CONFLICT' }`. Unknown errors are rethrown.
- One response shape for all errors: a machine-readable `code` plus optional `detail`. User-facing text is produced in the UI, not in the service.
- A side effect that is best-effort (cache clear, notification) must not fail the main operation. Report it in the result instead.

## 4. Config and secrets

- One module reads the environment (`env.ts`). It validates on startup and fails closed when a required value is missing. No other file touches `process.env`.
- `.env.example` lists every variable name with a safe dev default. `.env` is gitignored.
- Numbers in config files carry a comment saying why.

Authentication has its own file: [auth.md](auth.md). The short version: JWT + EdDSA + JWKS, OAuth 2.1 for non-browser clients, no bypass in dev, real dev tokens.

## 5. Database

- The schema is defined in one place. Change it there, generate the migration, read the SQL, and commit it together with the code that uses it.
- Migrations are forward-only and numbered. Change in two steps so the previous release keeps working: add as nullable first, remove later.
- Multi-tenant: every table carries the tenant id, and services receive a tenant-scoped handle instead of the raw connection. Derive the tenant from a trusted source (the `Host` header or the session), never from a header a client can forge. Add a check that fails when a table lacks the tenant column.
- Money is an integer in the smallest unit. Times cross the wire as epoch milliseconds or ISO strings, chosen once.
- Put invariants in the database (constraints, unique indexes) as well as in code.

## 6. TypeScript baseline

```json
{
  "compilerOptions": {
    "target": "ESNext", "module": "ESNext", "moduleResolution": "bundler",
    "strict": true, "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true, "isolatedModules": true,
    "noEmit": true, "skipLibCheck": true, "types": ["bun"]
  }
}
```

In a monorepo this is `tsconfig.base.json` and every workspace extends it. Pin shared dependency versions once in the root catalog so there is a single copy of each framework package.

## 7. Testing

- Pure logic lives where it can be tested without a server or a browser.
- Service tests run against a real database from `docker compose`. They skip with a clear message when it is down, and a strict flag makes them fail instead for CI.
- A check command must never hide a failure: no piping test output through something that turns a failing exit code into success.
- One command runs everything a reviewer needs: `bun run check` (format check, lint, types, tests).

## 8. Developer experience

- One command starts everything: `bun run dev` brings up local infrastructure, runs migrations and seed, and starts the apps.
- Local infrastructure uses non-default ports so it does not clash with other projects.
- Seed data is committed only if it is synthetic or scrubbed by a script. Never hand-edit a dump into the repo.
- Root scripts: `dev`, `build`, `check`, `test`, `lint`, `fmt`, `db:generate`, `db:migrate`, `db:seed`.

## 9. Decisions are written down

- `docs/architecture.md` holds the current structure and rules.
- `docs/plans/YYYY-MM-DD-<topic>.md` holds one decision or feature plan each: the current state, the decision, and the alternatives that were rejected with the reason.
- When a doc and the code disagree, fix one of them in the same change. Stale docs mislead the next person and the next agent.
- Keep the agent guide (`CLAUDE.md` / `AGENTS.md`) short: commands, the layer rules, and links. Changelogs and workaround lists go in their own files.

## `docs/architecture.md` template

```md
# Architecture

Status: draft | accepted · Date: YYYY-MM-DD · Owner: <name>

## What this is
One paragraph: the product, who uses it, the first thing that must work.

## Requirements that shaped the design
- <requirement> → <consequence>

## Stack
| Concern | Choice | Why |
|---|---|---|

## Folder structure
<tree with one comment per folder>

## Rules
- Dependency direction: ...
- Where a new feature goes: ...
- Errors: ...
- Auth: token format, key handling, how to get a dev token
- Config: ...
- Database changes: ...

## Assumptions
Defaults chosen without an explicit requirement.

## Left out on purpose
| Not included | Add it when |
|---|---|

## Commands
dev · check · test · build · db:generate · db:migrate
```
