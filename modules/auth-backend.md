# auth-backend — Spec

<!-- Extracted from komorebi server/src/auth (2026-08-15), per owner decision:
     OTP for both email and SMS (the source's magic-link flow was dropped). -->

## Problem

A backend needs passwordless, invite-only authentication with sessions that
work for native and web clients alike, plus an OAuth 2.1 provider so
third-party developer apps can access the public API with user consent. This
module wires Better Auth into the NestJS app: OTP sign-in over email and SMS,
sliding Bearer-token sessions, a signup gate backed by an allowlist table, a
request guard that feature modules apply to protected routes, and first-party
revocation of connected OAuth apps.

## No-goals

- No passwords and no magic links — email OTP and SMS OTP are the only
  first-party sign-in methods.
- No social login (Apple/Google as upstream identity providers).
- No open self-serve signup — account creation is invite-only via the
  `allowlist` table; no admin UI for the allowlist (rows inserted by hand).
- No first-party refresh-token flow — the Bearer token IS the session.
- No login/consent UI — those pages live in the web app; this module only
  points the OAuth provider at them.
- No stateless-JWT verification path for the public API: revoking a connected
  app must be immediate, so access tokens are resolved by DB row lookup
  (deleted row = dead token). Integrators must not send a `resource` param on
  the token request (that mints a JWT with no row).

## Domain types

Added to `core/domain/auth.ts`:

```ts
export type UserRole = 'admin' | 'user';

/** What AuthGuard attaches to authenticated requests. */
export interface AuthedUser {
  id: string;      // uuid, FK target for every app table
  email: string;
  role: UserRole;
}

/** Emitted after a user row is created, whatever the sign-in method. */
export interface UserRegisteredEvent {
  userId: string;
  email: string | null;
  phoneNumber: string | null;
}
```

Added to `core/lib/hash.ts` (shared with the public-api module's token
lookup — must match the oauth-provider plugin's `storeClientSecret: 'hashed'`
digest):

```ts
import { createHash } from 'node:crypto';

/** SHA-256, unpadded base64url. */
export function hashClientSecret(secret: string): string {
  return createHash('sha256').update(secret).digest('base64url');
}
```

Consumes from `core/`: `EmailSender`, `SmsSender` (constitution providers —
Resend and Surge implementations live in `core/lib`), typed env config.

## Module map

- **Now**: OTP sign-in (email + SMS), sliding Bearer sessions, allowlist
  signup gate, `AuthGuard`, OAuth 2.1 provider (PKCE, JWKS, self-serve
  clients), connected-app revocation.
- **Next**: `users` module (profile; subscribes to `user.registered` for
  post-signup setup), `public-api` module (scope guard resolving access
  tokens via `hashClientSecret`).
- **Later**: open-signup policy, Redis rate-limit store (multi-instance),
  non-US SMS regions, magic-link sign-in as an optional variant.

## Contract

### Endpoints

**Module-owned** (NestJS, `{ data, error }` envelope):

| Method + path | Input (zod) | Output |
|---|---|---|
| `DELETE /api/oauth/connections/:id` | `z.object({ id: z.string().uuid() })` (params) | 204, empty body |

Requires a valid session (`AuthGuard`). Revokes the consent AND every
access/refresh token for its (user, client) pair in one transaction.

**Mounted** (Better Auth handler at `/api/auth/*`; shapes are plugin-owned,
NOT envelope-wrapped — the handler receives the raw request stream):

| Method + path | Purpose |
|---|---|
| `POST /api/auth/email-otp/send-verification-otp` | `{email, type:"sign-in"}` → send 6-digit code via `EmailSender` |
| `POST /api/auth/sign-in/email-otp` | `{email, otp}` → session; token in body + `set-auth-token` header |
| `POST /api/auth/phone-number/send-otp` | `{phoneNumber}` → send code via `SmsSender` |
| `POST /api/auth/phone-number/verify` | `{phoneNumber, code}` → session |
| `GET /api/auth/get-session` | current session or null |
| `POST /api/auth/sign-out` | invalidate the session |
| `GET/POST /api/auth/oauth2/authorize` | OAuth 2.1 authorize; redirects to web login/consent pages |
| `POST /api/auth/oauth2/token` | code/refresh grants; PKCE S256 required |
| `GET /api/auth/oauth2/userinfo` | OIDC userinfo |
| `GET /api/auth/jwks` | JWKS (jwt plugin) |
| `POST /api/auth/oauth2/create-client` (+ `get-clients`, `update-client`, `rotate-secret`, `delete-client`) | session-authenticated self-serve client management; anonymous RFC-7591 `/oauth2/register` is disabled |

### Events

- **Emits**: `user.registered` → `UserRegisteredEvent` — after the user row
  is created. (Replaces the source project's direct cross-module call for
  post-signup setup; consumers do handle generation, welcome emails, etc.)
- **Listens**: none.

### Business rules

1. When a guarded route receives no valid Bearer session, then 401
   "Invalid or expired session".
2. When the session is valid, then `req.user = { id, email, role }` with
   `role` defaulting to `user` when absent.
3. When user creation runs with an identity that has no `allowlist` row,
   then reject 403 "Sign-ups are invite-only". All sign-up paths converge on
   this single user-create DB hook. `[VERIFY]` allowlist email matching is
   case-insensitive while phone matching is exact E.164 string equality.
4. When the matched allowlist row pairs an email with a phone, then both are
   stamped on the user at creation, so whichever method is used second
   resolves to the same account (account linking enabled).
5. When a user row is created, then `user.registered` is emitted.
6. When an OTP is issued (email or SMS), then it is 6 digits, expires in
   5 minutes, and allows 3 verification attempts (plugin defaults).
7. When an SMS OTP is sent, then the copy names the app and the expiry:
   "Your {APP_NAME} sign-in code is {code}. It expires in 5 minutes."
8. When the phone number fails {PHONE_REGION_VALIDATOR}, then the send is
   rejected.
9. When a phone-first sign-up verifies, then the user gets temp email
   `<phone>@{PHONE_TEMP_EMAIL_DOMAIN}` until the allowlist pairs a real one.
10. When either OTP-send endpoint exceeds 1 request/minute/IP, then 429.
    (`ponytail:` in-memory rate-limit store — fine for one instance; Redis
    when scaling horizontally.)
11. When a session is used, then it slides: {SESSION_TTL_DAYS}-day expiry,
    extended at most once per {SESSION_UPDATE_AGE_DAYS} day(s); inactive
    sessions lapse. The Bearer token IS the session.
12. When `DELETE /api/oauth/connections/:id` succeeds, then consent plus all
    access and refresh tokens for that (user, client) die in one
    transaction — 404 for an unknown consent, 403 for another user's, 204 on
    success. Revocation is immediate because token verification is
    row-backed.
13. When a user already owns {MAX_DEVELOPER_APPS} OAuth clients, then
    creating another is denied. (`ponytail:` the allow/deny hook surfaces the
    cap as the plugin's 401, not a 400 — acceptable while the portal shows
    the limit.)
14. When a third-party requests authorization, then scopes must be a subset
    of {OAUTH_SCOPES}, PKCE S256 is required, and anonymous client
    registration is off.
15. When a browser calls `/api/auth/*`, then CORS allows exactly
    `[WEB_ORIGIN, ...AUTH_TRUSTED_ORIGINS]`; the same list feeds Better
    Auth's `trustedOrigins` from one function so they never drift; the
    `set-auth-token` response header is exposed.
16. When client secrets are stored or access tokens resolved, then the
    digest is SHA-256 unpadded base64url on both sides (`hashClientSecret`).
17. When user rows are created, then ids are uuid. `[VERIFY]` kept so app
    tables can FK `users(id)` with uuid columns.
18. When any client supplies `role`, then it is ignored — the field is
    server-side only (`input: false`); the DB column is the source of truth.
19. `[VERIFY]` When Better Auth connects, it opens its own pg Pool from
    `DATABASE_URL`, separate from the ORM's pool (two pools, one database).

### Examples

Send email OTP:

```
POST /api/auth/email-otp/send-verification-otp
{ "email": "mia@example.com", "type": "sign-in" }
→ 200 { "success": true }
```

Verify email OTP (sign-in):

```
POST /api/auth/sign-in/email-otp
{ "email": "mia@example.com", "otp": "482913" }
→ 200  (set-auth-token: <bearer token>)
{ "token": "<bearer token>", "user": { "id": "5e0c…", "email": "mia@example.com" } }
```

Send SMS OTP:

```
POST /api/auth/phone-number/send-otp
{ "phoneNumber": "+14155550123" }
→ 200 { "message": "code sent" }
```

Verify SMS OTP:

```
POST /api/auth/phone-number/verify
{ "phoneNumber": "+14155550123", "code": "482913" }
→ 200 { "status": true, "token": "<bearer token>", "user": { "id": "5e0c…" } }
```

Non-allowlisted sign-up attempt (either channel, on verify):

```
→ 403 { "message": "Sign-ups are invite-only" }
```

Revoke a connected app:

```
DELETE /api/oauth/connections/9d2f6a1c-8f6e-4b1a-9c2d-1e5f7a3b4c5d
Authorization: Bearer <session token>
→ 204

(unknown id)              → 404 { "data": null, "error": { "message": "Consent not found" } }
(another user's consent)  → 403 { "data": null, "error": { "message": "Consent belongs to another user" } }
```

### Error table

| Error code | HTTP status | When it occurs |
|---|---|---|
| UNAUTHORIZED | 401 | Guarded route with a missing/invalid/expired session |
| FORBIDDEN | 403 | Sign-up identity not in the allowlist |
| BAD_REQUEST | 400 | Phone number fails the region validator; invalid/expired/exhausted OTP |
| TOO_MANY_REQUESTS | 429 | OTP send exceeding 1/minute/IP |
| NOT_FOUND | 404 | Revoke: consent id does not exist |
| FORBIDDEN | 403 | Revoke: consent belongs to another user |
| UNAUTHORIZED | 401 | OAuth client creation past the per-user cap (plugin ceiling, see rule 13) |

## Env vars

| Name | Purpose | Example | Required |
|---|---|---|---|
| `DATABASE_URL` | Postgres; Better Auth opens its own pool | `postgres://…` | yes |
| `BETTER_AUTH_SECRET` | Signing secret | 32+ random chars | yes |
| `BETTER_AUTH_URL` | Public base URL of this API | `https://api.example.com` | yes |
| `AUTH_WEB_ORIGIN` | Web origin hosting login/consent pages; first CORS origin | `https://example.com` | yes |
| `AUTH_EMAIL_FROM` | Sender for auth emails | `Name <noreply@example.com>` | no — defaults to {EMAIL_FROM_DEFAULT} |
| `AUTH_TRUSTED_ORIGINS` | Extra browser origins, comma-separated | `http://localhost:3000` | no |
| `RESEND_API_KEY` | core `EmailSender` (Resend) | `re_…` | yes |
| `SURGE_API_KEY` | core `SmsSender` (Surge) | `sk_…` | yes |
| `SURGE_ACCOUNT_ID` | core `SmsSender` (Surge) | `acct_…` | yes |

The Resend/Surge vars are consumed by the `core/lib` provider
implementations; they are declared here because this module introduces the
need for them.

## Integration surface

- **`AuthGuard`** — apply `@UseGuards(AuthGuard)` (exported by `AuthModule`)
  to any route needing a first-party session; read `req.user: AuthedUser`
  (type from `core/domain/auth.ts`).
- **`AUTH` token (Better Auth instance)** — mounting contract in `main.ts`:
  create Nest with `bodyParser: false`, mount
  `express.all('/api/auth/{*any}', toNodeHandler(auth))` BEFORE registering
  the JSON parser (Better Auth needs the raw stream), enable CORS with the
  rule-15 origin list and `exposedHeaders: ['set-auth-token']`.
- **`hashClientSecret`** (`core/lib/hash.ts`) — the public-api module resolves
  access tokens by hashed row lookup in `oauthAccessToken`; both sides must
  use this exact digest.
- **Events emitted**: `user.registered` — subscribe for post-signup setup
  (profile bootstrap, generated handles, welcome emails).
- **Tables owned**: `users`, `session`, `account`, `verification`,
  `allowlist`, `oauthClient`, `oauthAccessToken`, `oauthRefreshToken`,
  `oauthConsent`. App tables FK `users(id)` (uuid, `ON DELETE CASCADE`).
  Nothing else writes these tables except allowlist rows (inserted by hand)
  and the public-api module's read-only token lookups.

## Parameters

| Parameter | Meaning | Example |
|---|---|---|
| `APP_NAME` | User-facing product name in email/SMS copy | `Jizo` |
| `EMAIL_FROM_DEFAULT` | Fallback sender when `AUTH_EMAIL_FROM` is unset | `Jizo <noreply@jizo.app>` |
| `OAUTH_SCOPES` | Scope list the OAuth provider offers (`openid` + `offline_access` + project `read:*` scopes) | `openid, offline_access, read:profile, read:location, read:posts, read:stories` |
| `PHONE_REGION_VALIDATOR` | Accepted phone pattern (SMS registration is per-region) | US E.164: `^\+1[2-9]\d{9}$` |
| `PHONE_TEMP_EMAIL_DOMAIN` | Placeholder email domain for phone-first signups | `phone.jizo.app` |
| `SESSION_TTL_DAYS` | Sliding session expiry | `30` |
| `SESSION_UPDATE_AGE_DAYS` | Minimum age before the expiry is extended | `1` |
| `MAX_DEVELOPER_APPS` | Per-user OAuth client cap | `5` |
| `EMAIL_PALETTE` | Brand colors + font stacks for the OTP email (inline-styled table layout, light-only) | warm palette: bg `#FBF1E3`, accent `#C2410C`, … |

## Outside the framework

- Deploy config (Railway), DNS, and the web app hosting `/login` and
  `/oauth/consent` pages on `AUTH_WEB_ORIGIN`.
- Resend account with a verified sending domain; Surge account with US A2P
  registration for the SMS sender.
- Migrations for the owned tables (manual, per constitution — Better Auth
  does not run its own).
- Allowlist rows: inserted by hand in SQL; pairing email+phone in one row
  marks one person's identities.
- The `core/lib` `EmailSender`/`SmsSender` provider implementations.
