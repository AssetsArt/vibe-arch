---
name: vibe-arch
description: Bootstraps a new software project by gathering requirements first, then choosing the stack, folder structure and core architecture rules, and scaffolding it. Use when the user starts a new project or repo from scratch, asks "which framework/stack should I use", wants a folder structure or architecture for a landing page, storefront, back-office/management system, API, mobile app or monorepo, or says "ขึ้นโปรเจกต์ใหม่", "วางโครงสร้างโปรเจกต์", "เลือก framework", "init project", "scaffold".
---

# Vibe Arch

Decide the architecture of a new project before any code exists. Requirements come first; stack and folders follow from them.

Talk to the user in the language they write in. Keep technical terms in English.

## Workflow

Work through these in order. Do not scaffold before step 4 is confirmed.

1. **Look first.** Check the target directory. If it already has code, a `package.json`, `Cargo.toml` or docs, read them and treat them as constraints. This skill is for new projects; for an existing one, propose changes instead of replacing its structure.
2. **Gather requirements.** Follow [requirements.md](requirements.md). Ask only what the user has not already said, in one batch, with a default beside each question so they can answer "ok" to all of them.
3. **Classify and choose.** Pick the project type and size tier, then the stack from [stack-selection.md](stack-selection.md) and the layout from [structures.md](structures.md).
4. **Present the proposal and wait for a yes.** Use the format below. Keep it to one screen.
5. **Scaffold.** Run the official create command for the chosen framework rather than hand-writing boilerplate, then shape the folders to match the layout. Apply [core-rules.md](core-rules.md).
6. **Write it down.** Create `docs/architecture.md` in the new project (template in [core-rules.md](core-rules.md)) and a short agent guide (`CLAUDE.md` or `AGENTS.md`) that links to it.
7. **Prove it runs.** Install, type-check, run the dev server or tests once, and report the real output.

## Non-negotiable defaults

- **TypeScript means Bun.** Bun is the runtime, package manager, script runner and test runner. Never propose npm, pnpm, yarn or Node as the first choice. If a hard constraint forces Node at runtime (a host or library that cannot run on Bun), say which constraint, keep Bun as the package manager, and record it in `docs/architecture.md`.
- **TypeScript tooling is fixed.** `bun test` for tests, oxlint and oxfmt for lint and format, and one `bun run check` gate. Set up in the first commit; see the baseline in [structures.md](structures.md).
- **Bun built-ins before libraries.** Check the Bun runtime docs before adding a dependency, for example `Bun.Image` before sharp. See the table in [stack-selection.md](stack-selection.md).
- **Pages are routes.** A React SPA gets React Router in the first commit. Never switch pages with `if` or a state variable in the app shell; every screen has a URL in one route table. See [stack-selection.md](stack-selection.md).
- **Auth is real from day one.** Recommend JWT signed with EdDSA, keys published as JWKS, and OAuth 2.1 for third-party, CLI and agent clients. Never bypass or stub authentication during development: build the full path and issue real dev tokens. See [auth.md](auth.md).
- **Smallest structure that fits.** One app in one folder until there are two deployables or logic shared by two apps. Only then move to a monorepo.
- **Pin, then check the docs.** Framework APIs move quickly. Read the current official docs for the chosen framework before scaffolding, and pin exact versions for anything pre-1.0, alpha or rc.
- **Every choice has a reason.** If a choice cannot be traced to a stated requirement, it is a default: label it as an assumption.

## Proposal format

```
Project type: <type> · Size: <single app | monorepo>
Stack: <runtime> · <framework> · <API> · <DB + ORM> · <auth> · <styling> · <tests> · <lint/format> · <deploy>
Why: one line per choice, tied to a requirement
Assumptions: defaults I picked that you did not state
Folder structure: tree, 2-3 levels, one comment per folder
Core rules: the 4-6 rules from core-rules.md that apply here
Not included yet: what was left out on purpose, and what would trigger adding it
```

## Reference files

- [requirements.md](requirements.md): the intake questions and how answers map to a project type
- [stack-selection.md](stack-selection.md): which framework, database and tooling to pick, and when
- [structures.md](structures.md): folder layouts per project type
- [auth.md](auth.md): the recommended auth design, the no-bypass rule and dev token issuing
- [core-rules.md](core-rules.md): layering, errors, config, database, testing rules and the `docs/architecture.md` template
