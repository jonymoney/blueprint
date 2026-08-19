# auth-backend — Human docs

> Companion to [auth-backend.md](auth-backend.md). Reading only — generation
> uses the spec exclusively.

## What it does, in plain language

Passwordless, invite-only sign-in for a NestJS backend, built on Better Auth.
Users sign in with a 6-digit code sent to their email (Resend) or phone
(Surge SMS) — no passwords, no magic links. Only people already in an
`allowlist` table can create an account; everyone else gets "Sign-ups are
invite-only". A successful sign-in returns a Bearer token that *is* the
session: it slides forward as you use it and dies after 30 days idle
(configurable).

Feature modules protect routes with the exported `AuthGuard` and read
`req.user` (id, email, role). When a new user is created, the module emits a
`user.registered` event so other modules can react (welcome email, profile
setup) without being called directly.

It's also an OAuth 2.1 provider: third-party developer apps can request user
consent and access the public API. Users can revoke a connected app at any
time, and revocation is instant because tokens are checked against DB rows —
delete the row, the token is dead.

## Dependency map

```mermaid
flowchart LR
  subgraph consumers["Feature modules"]
    FM["any module"]
  end
  subgraph auth["src/modules/auth-backend"]
    G["AuthGuard"]
    BA["Better Auth instance<br/>(AUTH token)"]
    RC["RevokeController<br/>DELETE /api/oauth/connections/:id"]
  end
  subgraph core["src/core"]
    ES["EmailSender (Resend)"]
    SS["SmsSender (Surge)"]
    H["hashClientSecret()"]
    ENV["typed env config"]
  end
  DB[("Postgres<br/>users · session · allowlist · oauth*")]
  EV(("user.registered"))

  FM -- "@UseGuards(AuthGuard)" --> G
  FM -. "subscribe" .-> EV
  BA --> ES & SS & ENV
  BA --> DB
  G --> BA
  RC --> DB
  auth -- "emits" --> EV
  H -. "same digest used by<br/>public-api token lookup" .-> DB
```

## Sign-in flow (email OTP — SMS is identical with phone/Surge)

```mermaid
sequenceDiagram
  actor U as User
  participant BA as Better Auth (/api/auth/*)
  participant R as EmailSender (Resend)
  participant DB as Postgres

  U->>BA: POST /email-otp/send-verification-otp {email}
  BA->>DB: store OTP (6 digits, 5 min, 3 attempts)
  BA->>R: send code email
  U->>BA: POST /sign-in/email-otp {email, otp}
  BA->>DB: verify OTP
  alt new user
    BA->>DB: allowlist row for email?
    alt not allowlisted
      BA-->>U: 403 Sign-ups are invite-only
    else allowlisted
      BA->>DB: create user (email + paired phone if any)
      BA--)BA: emit user.registered
    end
  end
  BA->>DB: create session
  BA-->>U: 200 {token, user} + set-auth-token header
```

## Guarded request

```mermaid
sequenceDiagram
  actor C as Client
  participant FM as Feature module route
  participant G as AuthGuard
  participant DB as Postgres

  C->>FM: GET /api/anything (Authorization: Bearer …)
  FM->>G: canActivate
  G->>DB: resolve session by token
  alt invalid / expired
    G-->>C: 401 Invalid or expired session
  else valid
    G->>G: req.user = {id, email, role}<br/>slide expiry (max 1×/day)
    FM-->>C: 200 {data, error} envelope
  end
```

## Third-party app (OAuth 2.1) + revocation

```mermaid
sequenceDiagram
  actor Dev as Third-party app
  actor U as User
  participant BA as Better Auth
  participant W as Web app (login/consent pages)
  participant RC as RevokeController
  participant DB as Postgres

  Dev->>BA: GET /oauth2/authorize (PKCE S256, scopes ⊆ allowed)
  BA->>W: redirect to /login → /oauth/consent
  U->>W: approve
  W->>BA: consent granted
  BA-->>Dev: code → POST /oauth2/token → access+refresh tokens (DB rows)
  Dev->>BA: API calls with access token (row lookup, hashed)

  Note over U,DB: later — user disconnects the app
  U->>RC: DELETE /api/oauth/connections/:id (Bearer session)
  RC->>DB: delete consent + all tokens (one transaction)
  RC-->>U: 204
  Dev->>BA: next API call
  BA-->>Dev: 401 (row gone — revocation is immediate)
```

## What you must set up (digest)

- **Env vars**: `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`,
  `AUTH_WEB_ORIGIN`, `RESEND_API_KEY`, `SURGE_API_KEY`, `SURGE_ACCOUNT_ID`
  (+ optional `AUTH_EMAIL_FROM`, `AUTH_TRUSTED_ORIGINS`).
- **Outside the module**: web pages for `/login` and `/oauth/consent`,
  verified Resend domain, Surge A2P registration, manual migrations for the
  owned tables, allowlist rows inserted by hand.
- **To consume it**: apply `@UseGuards(AuthGuard)`, subscribe to
  `user.registered`, and follow the `main.ts` mounting contract (raw body
  for `/api/auth/*`). Full details: spec's *Integration surface*.
