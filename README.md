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

## What the skill does

1. Checks the target directory for anything that already exists.
2. Asks for requirements in one round, with a default for every question: management system, landing page, storefront, API, mobile app or platform.
3. Chooses the stack. TypeScript always starts with Bun; the framework comes from Brust, Next.js, Nuxt, React, Elysia, Drizzle and Tailwind, depending on the requirements.
4. Proposes the folder structure and architecture rules, and waits for confirmation before writing code.
5. Scaffolds the project, writes `docs/architecture.md`, and runs it to show that it works.

## TypeScript baseline

Every TypeScript project gets the same tooling in its first commit: `bun test`, oxlint, oxfmt, and a single `bun run check` gate. Before adding a dependency, the skill checks whether Bun already has a built-in for the job, for example `Bun.Image` before sharp.

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
