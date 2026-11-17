# Authentication

Applies to every project that has login, an API with callers, or agent/CLI access. A landing page with no login skips this file.

## Recommended design

| Part | Choice |
|---|---|
| Token format | JWT |
| Signature | EdDSA (Ed25519). The only accepted algorithm. |
| Public keys | Published as a JWKS endpoint, referenced by `jwks_uri` in the server metadata |
| Third-party, CLI and AI-agent clients | OAuth 2.1: authorization code + PKCE, refresh token rotation, no implicit or password grant, published authorization-server metadata |
| Browser sessions | The session JWT in an HttpOnly, Secure, SameSite cookie, backed by a server-side session row |
| Library | `jose`. Never hand-write JWS encoding or verification. |

If the user asks for something else (a hosted identity provider, an auth library), use it, and keep the rules below that still apply: no bypass, real tokens in dev.

## Build the full path from the first commit

Never stub authentication. Do not add a flag, header, hard-coded user or `if (dev) return admin` that skips verification, not even "temporarily". A bypass means the real path is untested until production, and bypasses get shipped.

The first protected route goes in together with signing, verification, the JWKS endpoint and the dev token issuer below. If that is too much for the first scaffold, leave the routes unprotected and unreleased, and say so under "Left out on purpose". Do not fake it.

## Rules

1. **A valid signature proves who issued the token, not that it is still allowed.** On every request, after verifying, read the session or grant row and the user row. Revoking a grant or disabling a user takes effect at once, even while the token's signature is still valid.
2. **The verifier decides the algorithm.** Accept `EdDSA` only. Never read `alg` from the token to choose how to verify.
3. **One key set per token purpose.** Access token, refresh token, browser session and signed links each get their own keys, their own `typ` header (for example `at+jwt`, `rt+jwt`) and a maximum lifetime. A door accepts only the purposes it names, so a token minted for one use cannot be replayed at another.
4. **Required claims:** `iss`, `aud`, `sub`, `exp`, `iat`, `jti`, plus the tenant id in multi-tenant systems. Check all of them. Allow about 60 seconds of clock skew. Reject tokens over a size cap before parsing.
5. **Short access tokens, single-use refresh tokens.** Access tokens live for minutes. A refresh token is rotated on every use and expires when idle.
6. **Key lifecycle:** `next` (published in JWKS before it signs) → `active` (the only signing key) → `retired` (still verifies until its longest token has expired) → deleted. Rotate lazily when signing, inside a transaction, so no scheduler is needed.
7. **Private keys are sealed at rest** with a key that lives in the deploy secrets and is backed up. Losing it means no new tokens until every key is rotated.
8. **Failure detail stays on the server.** Log why verification failed (`expired`, `unknown_kid`, `signature`). Tell the client only 401 or 403.
9. **Tenant comes from a trusted source** (the `Host` header or the verified token), never from a header the client can set freely.

## Development: issue a real dev token

Dev differs from production in exactly one place: where the identity comes from. Everything after that is the real path.

- Provide a dev token issuer: a dev-only login page or endpoint (`POST /session/dev-issue { userId }`) for apps with a UI, and a script (`bun run dev:token --user <id>`) for API services. Both call the same function.
- The issuer signs with the real signing code and creates a real session or grant row. Every later request is verified exactly as in production.
- The identity is a real, active user from the dev database (seed data). No invented roles: a user whose role is not allowed gets the real "forbidden" response, which is the point of testing with it.
- The issuer is reachable only when all three hold: an explicit dev flag is set, the process is not in production mode, and the database is local. Otherwise the route answers 404. Put this check in one function (`devIssueAllowed()`); it is the only code that reads the dev flag.
- The user list for the dev login page closes its own door with the same check, so names cannot leak if a caller forgets.
- OAuth and token code never references the dev flag or `NODE_ENV`. Add a test that greps for it.
- The dev orchestrator (`bun run dev`) generates the seal key into `.env` on first run. Signing keys and seal keys never go into committed seed data.

## Where it lives

```
server/                      # or packages/services/src in a monorepo
├── signing/
│   ├── keys.ts              # purposes, key set per purpose, lazy rotation
│   ├── jwt.ts               # sign, verify, jwks
│   └── seal.ts              # seal and unseal private keys
├── auth/
│   ├── session.ts           # issue and read sessions; reads the user row every request
│   ├── verify.ts            # request → identity, or a typed failure
│   ├── dev-env.ts           # devIssueAllowed(): the one dev gate
│   └── dev-issue.ts         # pick a real user and issue a real session
└── oauth/                   # authorize, token, metadata, grants, rate limits
api/plugins/auth.ts          # cross-cutting plugin: 401 / 403 / 503 mapping
scripts/dev-token.ts         # CLI issuer for API-only projects
```

HTTP surface: `GET /.well-known/oauth-authorization-server`, `GET /oauth/jwks` (cache about 5 minutes), `/oauth/authorize`, `/oauth/token`, and the dev-only `POST /session/dev-issue`.

## Tests to write with it

- A token signed with another algorithm, an unknown `kid`, a wrong `typ`, a wrong audience, or an expired `exp` is rejected.
- A token for one purpose is rejected at a door for another.
- A script using `jose` directly, with keys fetched from the JWKS endpoint, verifies a token the app issued.
- Revoking the grant or disabling the user returns 401 while the signature is still valid.
- The dev issuer returns 404 when any of its three conditions is false.
- A retired key still verifies until its window closes; a rotated-in key signs afterwards.
