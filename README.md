# vibe-arch

An agent skill for starting a new project: it gathers requirements first, then helps choose the stack, lay out the folder structure and set the core architecture rules.

## Install

```bash
# From GitHub
npx skills add AssetsArt/vibe-arch

# Install for every project, for one agent
npx skills add AssetsArt/vibe-arch -g -a claude-code

# Try it from a local checkout
npx skills add ./vibe-arch
```

## Skills in this repo

| Skill | Use it when |
|---|---|
| `vibe-arch` | Starting a new project, asking which framework to use, or asking for a project structure |

## Usage

After installing, describe the project you want to start. The agent loads the skill on its own when the request is about a new project, a framework choice or a project structure. In agents that support slash commands you can also call it by name: `/vibe-arch`.

Example prompts:

```text
Start a new project: a back office for managing orders and stock, with staff roles.
```

```text
I need a landing page for a product launch. Which stack should I use?
```

```text
/vibe-arch API service for a mobile app, PostgreSQL, clients log in with OAuth
```

```text
ขึ้นโปรเจกต์ใหม่ เป็นระบบจัดการร้านค้า มีหน้าร้านและหลังบ้าน
```

What a session looks like:

```text
You:   Start a new project: a back office for managing orders and stock.

Agent: The folder is empty. A few questions, each with a default. Reply "ok" to accept all.
       1. Users: internal staff, under 100?            (default: yes)
       2. Own database, or an existing one?            (default: new PostgreSQL)
       3. Roles: admin and staff?                      (default: yes)
       4. Other clients: CLI, services, AI agents?     (default: none)
       5. Deploy target?                               (default: Bun process in a container)
       6. React or Vue? Pre-1.0 frameworks acceptable? (default: React, stable only)
       7. First thing that must work?                  (default: log in and list orders)

You:   ok, but we use Vue

Agent: Project type: Management system · Size: single app
       Stack: Bun · Nuxt · Drizzle + PostgreSQL · JWT/EdDSA + JWKS · Tailwind
              · bun test · oxlint + oxfmt · container
       Why: Vue team → Nuxt; staff roles → session auth with a dev token issuer; ...
       Assumptions: PostgreSQL, container deploy
       Folder structure: ...
       Core rules: ...
       Not included yet: queue, cache, i18n (add when ...)
       Proceed?

You:   yes

Agent: (scaffolds, writes docs/architecture.md, runs `bun run check`, reports the output)
```

Tips:

- Say what you already know in the first message. The skill only asks about what is missing.
- Name a framework if you have one in mind. The skill uses it and states the trade-off once.
- Reply "ok" to take every default. They are listed under "Assumptions" in the proposal.

## What the skill does

1. Checks the target directory for anything that already exists.
2. Asks for requirements in one round, with a default for every question: management system, landing page, storefront, API, mobile app or platform.
3. Chooses the stack. TypeScript always starts with Bun; the framework comes from Brust, Next.js, Nuxt, React, Elysia, Drizzle and Tailwind, depending on the requirements.
4. Proposes the folder structure and architecture rules, and waits for confirmation before writing code.
5. Scaffolds the project, writes `docs/architecture.md`, and runs it to show that it works.

## TypeScript baseline

Every TypeScript project gets the same tooling in its first commit: `bun test`, oxlint, oxfmt, and a single `bun run check` gate. Before adding a dependency, the skill checks whether Bun already has a built-in for the job, for example `Bun.Image` before sharp.

## Routing

A React SPA gets React Router in its first commit. Pages are routes in one route table with real URLs; the app shell never switches pages with `if` or a state variable.

## Authentication

The skill recommends JWT signed with EdDSA, public keys published as JWKS, and OAuth 2.1 for CLI, service and AI-agent clients. It never bypasses authentication during development: the full path is built from the first protected route, and a dev token issuer signs real tokens for real seeded users.

## Repository layout

```
skills/vibe-arch/
├── SKILL.md              # main workflow
├── requirements.md       # intake questions
├── stack-selection.md    # framework and tooling choices
├── structures.md         # folder structures per project type
├── auth.md               # auth design, no-bypass rule, dev tokens
└── core-rules.md         # architecture rules and the doc template
```
