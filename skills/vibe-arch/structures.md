# Folder structures

Pick the smallest layout that fits. Rename folders to match the chosen framework's conventions (for example Next.js `app/`, Nuxt `pages/` and `server/`), but keep the same separation of concerns.

Naming used throughout:

- React components and route pages: `PascalCase.tsx`. Logic modules: `kebab-case.ts`.
- Tests: `*.test.ts` beside the code or in `tests/`. End-to-end: `*.e2e.ts`.
- Generated files: `*.generated.ts`, with a "do not edit" header, excluded from lint and format.
- Named exports everywhere. Default exports only where the framework requires them (route components).

## TypeScript baseline (every TS project)

Every TypeScript layout below starts with these root files and scripts. Set them up in the first commit, before feature code.

```
<project>/
├── package.json          # scripts below; exact versions for pre-1.0 deps
├── bun.lock              # committed
├── tsconfig.json         # strict baseline from core-rules.md
├── .oxlintrc.json        # lint rules + import boundaries
├── .oxfmtrc.json         # format settings
├── .env.example          # variable names with safe dev defaults
└── .gitignore            # .env, dist, node_modules
```

```json
{
  "scripts": {
    "dev": "<framework dev command>",
    "build": "<framework build command>",
    "test": "bun test",
    "typecheck": "tsc --noEmit",
    "lint": "oxlint",
    "fmt": "oxfmt",
    "fmt:check": "oxfmt --check",
    "check": "bun run fmt:check && bun run lint && bun run typecheck && bun test"
  }
}
```

- **`bun run check` is the one gate.** It runs format check, lint, types and tests, and stops at the first failure. Run it before every commit and in CI. Do not pipe it through anything that can hide a failing exit code.
- **Tests use `bun test`.** No Jest or Vitest. Test files are `*.test.ts`.
- **Lint with oxlint, format with oxfmt.** No ESLint or Prettier. Starting `.oxlintrc.json`:

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["typescript", "react", "import", "unicorn"],
  "categories": { "correctness": "error", "suspicious": "warn" },
  "ignorePatterns": ["**/dist/**", "**/*.generated.ts", "**/*.generated.tsx"]
}
```

  Drop the `react` plugin for non-React projects. Use `no-restricted-imports` to enforce layer boundaries, such as allowing the database driver only inside the data package.
- **In a monorepo** these files live at the root only. Each workspace has a `tsconfig.json` that extends the base and a `typecheck` script; the root runs `bun run --filter '*' typecheck`.
- **Bun built-ins before libraries.** Before adding a dependency, check whether Bun already does the job. See the table in [stack-selection.md](stack-selection.md).

## 1. Landing page (single app, static)

```
my-site/
├── package.json  bun.lock  tsconfig.json
├── routes.tsx            # route table (or the framework's pages/ folder)
├── app.css               # Tailwind import + @theme tokens
├── pages/                # one component per route
├── components/           # sections and shared UI
│   └── ui/               # small primitives (Button, Container)
├── content/              # copy, blog posts, data as files
├── lib/                  # pure helpers (formatting, SEO metadata)
├── public/               # images, fonts, favicon
└── docs/architecture.md
```

No `server/`, no database, no API. Add them only when a form or dynamic data appears.

## 2. Full-stack web app (single app)

For a management system, storefront or content site that is one deployable.

```
my-app/
├── package.json  bun.lock  tsconfig.json  .env.example
├── routes.tsx            # single route table
├── actions.ts            # composition root for server actions / API
├── app.css
├── pages/                # thin route components: layout + wiring only
├── components/           # shared UI
│   └── ui/               # design-system primitives
├── features/             # one folder per feature
│   └── <feature>/
│       ├── <Feature>Page.tsx   # screens for this feature
│       ├── model.ts            # request/response schemas
│       ├── ops.ts              # state changes, pure where possible
│       └── ops.test.ts
├── loaders/              # server-side reads, one per screen
├── server/               # server-only code; never imported by the browser bundle
│   ├── services/         # business logic, HTTP-agnostic, throws typed errors
│   ├── signing/          # JWT keys, sign, verify, jwks (see auth.md)
│   ├── auth/             # sessions, verify, dev token issuer
│   ├── api/              # HTTP modules: controller + model per feature
│   ├── db/               # schema.ts, client.ts, migrations/
│   └── env.ts            # the one place that reads process.env
├── lib/                  # pure logic shared by server and client
├── tests/                # integration and e2e
├── scripts/              # seed, codegen, dev helpers
├── docker/compose.yml    # local database only
└── docs/
    ├── architecture.md
    └── plans/            # one dated file per significant decision or feature
```

If the app has no database of its own and only fronts another backend, drop `server/db/` and keep one module (`server/backend-client.ts`) as the only place that calls that backend.

## 3. Platform (monorepo)

Use when there are two or more deployables or shared logic.

```
my-platform/
├── package.json          # workspaces + version catalog + all root scripts
├── bun.lock  tsconfig.base.json  .env.example
├── apps/                 # deployables; each has the layout of section 2, minus server/
│   ├── web/              # public site or storefront
│   ├── admin/            # back office
│   └── worker/           # background job consumer (plain Bun process)
├── packages/
│   ├── core/             # pure domain: types, rules, validators. No I/O, no framework.
│   ├── ui/               # shared UI primitives
│   ├── theme/            # CSS tokens only
│   ├── db/               # schema, connections, migrations. Only package that imports the driver.
│   ├── services/         # business logic. Depends on core and db. HTTP-agnostic.
│   ├── api/              # HTTP modules mounted by the apps. Depends on services.
│   └── jobs/             # job names, payload types, handlers
├── tools/                # codegen and dev orchestration scripts
├── docker/compose.yml
└── docs/
```

Dependency direction, one way only:

```
apps → api → services → db
apps → ui, theme, core
everything → core        core → nothing
```

Two kinds of shared package, kept apart:

- **Pure packages** (`core`, `ui`, `theme`): safe in the browser. No I/O.
- **Server packages** (`db`, `services`, `api`, `jobs`): must never reach the browser bundle. Enforce it with the framework's server-only setting and a lint rule that blocks the database driver outside `packages/db`.

Workspace packages point their `exports` at TypeScript source. No per-package build step.

## 4. API service

```
my-api/
├── package.json  bun.lock  tsconfig.json  .env.example
├── src/
│   ├── main.ts           # thin: read config, build app, listen
│   ├── app.ts            # composition root: mounts every module
│   ├── plugins/          # cross-cutting: auth, tenant, request id
│   ├── signing/  auth/  oauth/   # see auth.md
│   ├── modules/
│   │   └── <feature>/
│   │       ├── index.ts      # routes (controller)
│   │       ├── model.ts      # request/response schemas
│   │       └── service.ts    # business logic, typed errors
│   ├── db/               # schema.ts, client.ts, migrations/
│   ├── env.ts
│   └── lib/
├── scripts/dev-token.ts  # issues a real token for a seeded user (dev only)
├── tests/
└── docs/architecture.md
```

## 5. Mobile app with a shared core

```
my-mobile/
├── Makefile              # one entry point: setup, check, test, build, run
├── core/                 # Rust workspace: all non-UI logic
│   └── crates/
│       ├── domain/       # pure rules (money, time, permissions). No I/O.
│       ├── api/          # HTTP clients and DTOs, one file per backend area
│       └── core/         # exported surface: one Store per feature + backend traits
│                         #   with a real and a mock implementation
├── ios/                  # SwiftUI app
│   └── App/{Core,Design,Shell,Modules/<Feature>}/
├── android/              # Compose app
│   └── app/.../ui/{components,theme,shell,<feature>}/
└── docs/{design,decisions}/
```

Each feature Store exposes the same four things: `state()`, `subscribe()`, `unsubscribe()`, `dispatch(intent)`. The UI never calls HTTP and never parses server JSON. Mock data lives in the core, not in the UI. Component names are the same on both platforms.

## 6. Rust workspace (systems, CLI)

```
my-system/
├── Cargo.toml            # workspace + shared dependency versions
├── rust-toolchain.toml  deny.toml
├── crates/
│   └── <area>/<crate>/   # grouped by area: common, runtime, control, cli
├── tests/                # acceptance tests that need special privileges
└── docs/{design,plans}/
```

A binary crate's `main.rs` only parses arguments and calls into `lib.rs`. `common/` crates are pure and usable by all others. Wire types live in their own crate with JSON fixtures and fixture tests.
