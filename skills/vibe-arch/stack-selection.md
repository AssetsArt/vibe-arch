# Stack selection

Read the current official docs before scaffolding. Versions and create commands change; the links below are the source of truth, not this file.

## Runtime and language

| Situation | Choice |
|---|---|
| Any TypeScript or JavaScript project | **Bun** for runtime, package manager, scripts and tests |
| Infrastructure, proxies, CLIs with tight performance or safety needs, logic shared by native iOS and Android | **Rust** (Cargo workspace) |
| Native mobile UI | Swift/SwiftUI and Kotlin/Compose |

Bun comes first for TypeScript, always. Falling back to Node at runtime needs a named constraint, and Bun stays as the package manager.

## Bun built-ins before libraries

Before adding any dependency to a TypeScript project, check the [Bun runtime docs](https://bun.com/docs/runtime) for a built-in. Fewer dependencies means faster installs, fewer native-build problems and less to audit. Add the library only when the built-in lacks something the project needs, and write that gap down in `docs/architecture.md`.

| Need | Bun built-in first | Instead of |
|---|---|---|
| Image resize, convert, metadata, placeholders | [`Bun.Image`](https://bun.com/docs/runtime/image) | sharp, jimp |
| Tests, mocks, coverage | `bun test` | Jest, Vitest |
| Run TypeScript, watch mode | `bun`, `bun --watch` | ts-node, tsx, nodemon |
| Load `.env` | Built in (`--env-file`) | dotenv |
| HTTP server, routing, WebSocket | `Bun.serve` | express, ws |
| PostgreSQL / MySQL client | `Bun.sql` | pg, postgres, mysql2 |
| SQLite | `bun:sqlite` | better-sqlite3 |
| Redis | `Bun.redis` | ioredis |
| S3-compatible storage | `Bun.s3` | AWS SDK |
| Password hashing | `Bun.password` | bcrypt, argon2 |
| Hashing | `Bun.hash`, `Bun.CryptoHasher` | hash libraries |
| Shell scripts | `Bun.$` | execa, shelljs |
| File I/O, glob | `Bun.file`, `Bun.write`, `Bun.Glob` | fs-extra, glob, fast-glob |
| YAML, TOML, JSON5, JSONL, Markdown, XML | Built-in parsers | js-yaml, toml, marked |
| Cookies, CSRF | `Bun.CookieMap`, `Bun.CSRF` | cookie, csurf |
| Scheduled work | `Bun.cron` | node-cron |
| HTML rewriting | `HTMLRewriter` | cheerio |
| Tar and zip archives | `Bun.Archive` | tar, archiver |
| Semver, colours, UUIDs | `Bun.semver`, `Bun.color`, `Bun.randomUUIDv7` | semver, chalk, uuid |
| Secrets in the OS keychain (CLI tools) | `Bun.secrets` | keytar |

How to apply it:

- Read the built-in's docs page for its limits before committing to it. Built-ins can be narrower than the library. For example, `Bun.Image` rotates in 90-degree steps only, and its HEIC and AVIF support depends on the operating system, so check the formats against the deploy target (usually Linux).
- With Drizzle, use its Bun driver for the chosen database instead of installing a separate client.
- The framework comes first: inside a full-stack framework, use its server and routing, not `Bun.serve` directly.
- Built-ins arrive with new Bun releases. If one is missing, check the installed version (`bun --version`) against the docs before reaching for a library.

## Web framework

| Framework | Pick it when | Watch out for |
|---|---|---|
| [Brust](https://brust.dev/) (React compiled ahead of time, Rust server, Bun loaders, Elysia actions) | React team, Bun-first, first-load speed matters, the team accepts a pre-1.0 framework. Good for storefronts, content sites, back offices and static marketing sites (static build target). | Pre-1.0. Pin the exact version, read release notes before upgrading, and keep a `docs/framework-issues.md` log of workarounds. |
| [Next.js](https://nextjs.org/) | React team that needs a stable, widely known framework, a large ecosystem of integrations, or a host built around it. | Run and install it with Bun. Keep server-only code out of client components. |
| [Nuxt](https://nuxt.com/) | The team writes Vue. | Same layering rules apply; use Bun to install and run. |
| Plain [React](https://react.dev/) SPA (Vite) | Internal tool behind login where SEO and first-load speed do not matter, talking to a separate API. | No SSR. Needs its own API service. |

Decision order:

1. Vue team: Nuxt.
2. React team, pre-1.0 acceptable: Brust.
3. React team, stable only: Next.js.
4. Internal SPA with an existing API: React + Vite.

If the user names a framework, use it. State the trade-off once and move on.

## API layer

- **Inside a full-stack framework**: use its own server functions, loaders or actions. Do not add a second HTTP server.
- **Standalone API on Bun**: [Elysia](https://elysiajs.com/), with its typed client (Eden) so the frontend gets request and error types without code generation.
- **Standalone API in Rust**: axum.
- Validate every request body at the boundary with the framework's schema tool (TypeBox in Elysia). Unknown fields are rejected or stripped on purpose, never by accident.

## Database

| Need | Choice |
|---|---|
| ORM for TypeScript | [Drizzle](https://orm.drizzle.team/) with drizzle-kit migrations |
| New relational database, no constraint | PostgreSQL |
| An existing MySQL or MariaDB must be used | Stay on it; Drizzle supports it |
| Small app, single node, or edge deployment | SQLite (Bun's built-in driver, or D1 on Cloudflare) |
| Cache, sessions, rate limits | Redis-compatible store, added only when needed |
| Background jobs | A queue (NATS JetStream or similar), added only when there is real asynchronous work |

No database for a landing page. Content lives in files.

## Authentication

| Need | Choice |
|---|---|
| Any login or protected API | JWT signed with EdDSA (Ed25519) using `jose`, public keys served as JWKS |
| Browser users | Session JWT in an HttpOnly cookie plus a server-side session row |
| Other services, CLIs, AI agents (MCP) | OAuth 2.1: authorization code + PKCE, rotating refresh tokens, published metadata |
| Local development | A dev token issuer that signs real tokens for real seeded users. No bypass. |

Details and rules are in [auth.md](auth.md).

## Styling and UI

- [Tailwind CSS](https://tailwindcss.com/), configured CSS-first with design tokens in a `@theme` block in one CSS file.
- Headless primitives (Radix or the Vue equivalent) for accessible dialogs, menus and selects. Write the visual components by hand on top.
- Icons from one icon set. Fonts self-hosted.

## Tooling

| Concern | Choice |
|---|---|
| Unit and integration tests | `bun test` |
| End-to-end tests | Playwright |
| Lint and format | oxlint + oxfmt, configured at the root |
| Type checking | `tsc --noEmit` with a strict base config (see core-rules.md) |
| The gate | `bun run check`: format check, lint, types, tests (see the baseline in structures.md) |
| Local infrastructure | `docker compose` for databases and queues only, on non-default ports |
| Rust | `cargo fmt`, `clippy -D warnings`, `cargo-deny` for licences |

## Mobile

| Situation | Choice |
|---|---|
| Both platforms, heavy business logic, offline state, or device hardware | Native SwiftUI + Compose over one shared Rust core exposed through UniFFI. The core owns state, HTTP and rules; the UI only renders. |
| Thin client over an existing API, TypeScript team, short deadline | Expo (React Native), installed and run with Bun |
| One platform only | Native for that platform |

## Recommended stack per project type

| Type | Default stack |
|---|---|
| Landing page | Bun · Brust static build (or Next.js static export / Nuxt generate) · Tailwind · static hosting. No database, no API. |
| Content site / storefront | Bun · Brust or Next.js or Nuxt with SSR and caching · Drizzle · Tailwind |
| Management system | Bun · Brust or Next.js or Nuxt · Elysia actions or framework server functions · Drizzle + PostgreSQL · Tailwind + headless primitives · JWT/EdDSA session auth with a dev token issuer |
| API service | Bun · Elysia · Drizzle + PostgreSQL · JWT/EdDSA + JWKS, OAuth 2.1 for external clients · `bun test` |
| Platform | Bun workspaces monorepo: `apps/*` for deployables, `packages/*` for shared code, version catalog at the root |
| Mobile app | See the mobile table |
| CLI / systems | Rust workspace, or a Bun CLI when it is glue code |
