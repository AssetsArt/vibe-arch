# Requirements intake

Ask in one batch. Skip anything the user already answered. Put a default beside every question so "ok" is a valid reply. If the user wants to move fast, take the defaults and list them under "Assumptions" in the proposal.

## The questions

1. **What is it?** Pick the closest:
   - Landing or marketing page (mostly static content)
   - Content site or storefront (public pages, SEO matters, data changes)
   - Management system, back office or dashboard (login, roles, CRUD, reports)
   - API or backend service only (other clients consume it)
   - Mobile app (iOS, Android, or both)
   - Platform (several of the above together)
   - CLI, library or systems software
2. **Who uses it and how many?** Public visitors, logged-in customers, internal staff. Rough scale: tens, thousands, millions.
3. **What data does it own?** None, read from an existing API, or its own database. If a database: is there an existing one that must be used?
4. **Login and permissions?** None, single role, several roles, or multi-tenant (many shops or organisations in one system). Who else needs access: other services, a CLI, AI agents? Is there an existing identity system to accept logins from? Default when login exists: JWT + EdDSA + JWKS, with OAuth 2.1 for non-browser clients (see auth.md).
5. **What else must it talk to?** Payment, email, chat, file storage, an existing backend, AI agents.
6. **Where does it run?** A server or container, Kubernetes, Cloudflare Workers, Vercel, static hosting. Default: a Bun process in a container.
7. **Team and constraints.** Languages the team knows (React or Vue), an existing design system, a deadline, and whether alpha-stage frameworks are acceptable. Default: React, no design system, stable frameworks only.
8. **What is the first thing that must work?** One sentence. This sets the scope of the first scaffold.

Extra questions, only when relevant:

- **SEO and first-load speed** matter? (public sites)
- **Languages** of the content and of the UI? (anything user-facing)
- **Realtime** updates or collaboration? Background jobs or scheduled work?
- **Offline use** or device hardware (camera, printer, scanner)? (mobile)
- **Money** involved? Then amounts are integers in the smallest unit from day one.

## From answers to project type

| Signal | Type |
|---|---|
| No login, no database, content changes rarely | Landing page |
| Public pages from a database, SEO matters | Content site / storefront |
| Login plus tables, forms and roles; SEO irrelevant | Management system |
| No UI of its own | API service |
| Runs on phones; needs store distribution or device hardware | Mobile app |
| Two or more of the above sharing data or logic | Platform (monorepo) |
| Runs in a terminal or as infrastructure | CLI / systems |

## Size tier

- **Single app**: one deployable. Flat folders inside one project.
- **Monorepo**: two or more deployables (for example site + admin + worker), or logic that two apps must share. Use workspaces with a central version catalog.

Do not start in a monorepo "for later". Moving one app into `apps/` when the second app appears takes minutes.

## Things to decide out loud

Say these in the proposal even if the user did not ask:

- What was left out on purpose (auth, queue, cache, i18n) and what would trigger adding it.
- Which answers were guesses.
- Any requirement that conflicts with another, such as "alpha frameworks not acceptable" together with a request for a pre-1.0 framework.
