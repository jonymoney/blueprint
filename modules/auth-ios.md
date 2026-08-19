# auth-ios — Spec

<!-- Extracted from komorebi ios/ (2026-08-19), on CONSTITUTION.ios.md
     (ADR-0001). Owner decision: OTP for both email and phone, matching
     modules/auth-backend.md — the source app's magic-link email flow and its
     deep-link handoff were deliberately dropped. Companion spec:
     modules/onboarding-ios.md (first-run gate). -->

## Problem

An iOS app on the ios constitution needs passwordless sign-in against the
auth-backend module and everything a Bearer session implies on-device: a
sign-in screen with email/phone OTP entry, secure token storage, automatic
auth headers on every request, detection of session death, and a sign-out
that leaves nothing of the previous account behind. This module is the
client half of `modules/auth-backend.md`; that spec is the source of truth
for the API shapes consumed here.

## No-goals

- No magic links and no deep-link/web-handoff callback — email and phone
  both sign in with a 6-digit OTP entered in-app.
- No social login (Sign in with Apple/Google) and no passkeys.
- No token refresh flow — the Bearer token IS the session (auth-backend
  rule 11); a dead session means re-authenticate.
- No onboarding content — the first-run gate is `modules/onboarding-ios.md`.
- No push-notification internals — this module only calls the app's
  registered lifecycle observers at sign-in/sign-out.
- No backend code — server behavior (allowlist gate, rate limits, OTP
  policy) lives in auth-backend.

## Domain types

Added to `Core/` (auth service, session plumbing):

```swift
protocol AuthManaging {
    var isAuthenticated: Bool { get }
    func requestEmailCode(email: String) async throws
    func verifyEmailCode(email: String, code: String) async throws
    func requestPhoneCode(phoneNumber: String) async throws
    func verifyPhoneCode(phoneNumber: String, code: String) async throws
    func signOut() async
}

/// Hooks the app registers for session boundaries. All calls best-effort;
/// sign-in/sign-out never fail because an observer did.
protocol AuthLifecycleObserving {
    /// After a session token is stored (e.g. upload a pending APNs token).
    func didSignIn() async
    /// Before the server session is revoked, while it can still authorize
    /// requests (e.g. unregister the device push token).
    func willSignOut() async
    /// After the local token is cleared — wipe account-scoped local state
    /// (database rows, image caches) so the next sign-in inherits nothing.
    func didSignOut() async
}

extension Notification.Name {
    /// Posted by AuthInterceptor when a 401 outside /api/auth/* proves the
    /// session dead. The app coordinator listens and evicts to sign-in.
    static let sessionExpired = Notification.Name("sessionExpired")
}

final class TokenStore {
    /// Keychain-backed. The Bearer session token — it IS the session.
    var sessionToken: String? { get set }
    var hasToken: Bool { get }
    func clear()
}
```

## Module map

- **Now**: sign-in feature (coordinator + VM + VC), `AuthManager`,
  `TokenStore`, `AuthInterceptor`, typed `AuthAPI` endpoints.
- **Next**: `onboarding-ios` (routing sibling ahead of this module),
  the app shell's routing contract (see Integration surface).
- **Later**: passkeys, magic-link variant, non-{PHONE_REGION} phone
  support, pre-wipe sync flush (see rule 12).

## Contract

### Endpoints

Consumed from the API — shapes owned by `modules/auth-backend.md`
(§Contract → Endpoints; not `{ data, error }`-wrapped):

| Method + path | Used for |
|---|---|
| `POST /api/auth/email-otp/send-verification-otp` | `{email, type:"sign-in"}` — request email code |
| `POST /api/auth/sign-in/email-otp` | `{email, otp}` → `{token}` — verify + session |
| `POST /api/auth/phone-number/send-otp` | `{phoneNumber}` — request SMS code |
| `POST /api/auth/phone-number/verify` | `{phoneNumber, code}` → `{token}` — verify + session |
| `POST /api/auth/sign-out` | revoke the current session server-side |

Sign-in responses carry the Bearer token in the JSON body (`token`).
Decoding of non-session responses is tolerant of Better Auth's varying
`{status}/{success}/{message}` shapes.

### Events

- **Emits**: `.sessionExpired` (NotificationCenter, no payload) — when a 401
  outside `/api/auth/*` proves the session dead.
- **Listens**: none. (The app coordinator subscribes; see Integration
  surface.)

### Business rules

1. When the app launches with no Keychain token, then the routing gate shows
   sign-in (after the onboarding gate — see onboarding-ios).
2. `[VERIFY]` When a token exists at launch, then route straight to the main
   app without validating it — optimistic launch; a stale/revoked token is
   evicted by the first 401 (rule 7).
3. When the user submits an email or phone, then request a code and advance
   the same screen to a code-entry step; the two channels are symmetric.
4. When phone input cannot be normalized to {PHONE_REGION} E.164, then
   reject locally with no network call.
5. When code verification succeeds, then store the response `token` in the
   Keychain and hand off to the main app.
6. When sign-in succeeds, then call every registered
   `AuthLifecycleObserving.didSignIn()` fire-and-forget — sign-in never
   waits on observers.
7. When any request outside `/api/auth/*` returns 401, then the session is
   dead: post `.sessionExpired`, sign out locally, evict to sign-in. A 401
   on `/api/auth/*` is an endpoint error (bad/expired code), never session
   death.
8. When any request is sent, then the interceptor attaches
   `Authorization: Bearer <token>` if a token exists.
9. When signing out, then in order: `willSignOut()` observers (while the
   session can still authorize), revoke server-side best-effort, always
   clear the Keychain token, then `didSignOut()` observers (local wipe).
   Sign-out always succeeds from the user's perspective.
10. When the code-entry step appears, then the code field uses
    `.oneTimeCode` content type for OS autofill and becomes first responder.
11. When an error occurs mid-flow, then restore the form for the current
    step and surface the message; every step is re-attemptable.
12. `[VERIFY]` When session expiry triggers the wipe, then unsynced local
    rows are lost. (`ponytail:` acceptable ceiling; upgrade path is a
    pre-wipe sync flush — see Later.)
13. When the view model publishes state, then it is one enum:
    `idle → loading → codeSent(identity) → signedIn | error(message)`.

### Examples

Phone sign-in, happy path (state transitions against live calls):

```
input  "(555) 555-0123"          → normalize "+15555550123"
POST /api/auth/phone-number/send-otp {"phoneNumber":"+15555550123"}
state  .codeSent("+15555550123") → code field, OTP autofill
POST /api/auth/phone-number/verify {"phoneNumber":"+15555550123","code":"482913"}
← 200 {"token":"kmb_…"}          → Keychain store, didSignIn(), state .signedIn
```

Email sign-in, wrong code then retry:

```
POST /api/auth/email-otp/send-verification-otp {"email":"mia@example.com","type":"sign-in"}
state  .codeSent("mia@example.com")
POST /api/auth/sign-in/email-otp {"email":"mia@example.com","otp":"000000"}
← 400                             → state .error("…"), code step restored
POST /api/auth/sign-in/email-otp {"email":"mia@example.com","otp":"482913"}
← 200 {"token":"kmb_…"}           → state .signedIn
```

Session death on an app request:

```
GET /api/posts …  ← 401
.sessionExpired posted → signOut() (local) → sign-in screen
```

### Error table

| Error | Surfaced as | When it occurs |
|---|---|---|
| Local validation | inline error state, no network call | phone input fails {PHONE_REGION} normalization; empty identity/code ignored |
| 400 from an `/api/auth/*` verify | error alert, step restored | wrong, expired, or attempt-exhausted OTP |
| 403 from an `/api/auth/*` verify | error alert | identity not allowlisted (auth-backend rule 3) |
| 429 from an `/api/auth/*` send | error alert | resend inside the 1/min window |
| 401 outside `/api/auth/*` | silent sign-out + eviction to sign-in | session expired or revoked |

## Env vars

Build-time configuration (per ADR-0001 this section covers build config):

| Name | Purpose | Example | Required |
|---|---|---|---|
| API base URL | per-environment `APIEnvironment` the shared `APIClient` targets | `https://api.example.com` | yes |
| Keychain service | namespace for the token entry | `{KEYCHAIN_SERVICE}` | yes |

## Integration surface

- **`AuthCoordinator`** — `start() -> UIViewController` + `onFinished`
  callback; the app coordinator mounts it when routing decides sign-in is
  needed and routes to the main app on `onFinished`.
- **`AuthManaging`** — resolved via `Container`; the routing gate reads
  `isAuthenticated`; settings/profile features call `signOut()`.
- **`.sessionExpired`** — the app coordinator must subscribe: on receipt,
  call `signOut()` (local cleanup; the server session is already dead) and
  evict to sign-in.
- **`AuthInterceptor`** — must be installed on the shared `APIClient` at
  bootstrap so every request gets the Bearer header and 401 detection.
- **`AuthLifecycleObserving`** — register observers at bootstrap for push
  registration (didSignIn/willSignOut) and account-data wipe (didSignOut:
  local database, image caches). The module ships with no observers.

## Parameters

| Parameter | Meaning | Example |
|---|---|---|
| `PHONE_REGION` | Phone normalization rule (must match auth-backend's `PHONE_REGION_VALIDATOR`) | US: 10 digits → `+1…`, 11 with leading `1` → `+…` |
| `KEYCHAIN_SERVICE` | Keychain service identifier for the token | `app.komorebi.ios` |
| `COPY` | Screen strings: title, per-channel subtitles, code-step title/subtitle, button labels | "Sign in" / "Enter your email to receive a code" / … |
| `LEGACY_KEYCHAIN_KEYS` | Old token keys removed on `clear()` (empty for new apps) | `auth/access_token, auth/refresh_token` |

## Outside the framework

- A deployed auth-backend instance (this module's server half), including
  its allowlist rows — a non-invited identity gets 403 on verify.
- The shared `APIClient`/`Endpoint` networking core, `Container` DI,
  `Theme` tokens, and the `@Setting` wrapper (ios constitution core).
- KeychainAccess (or equivalent) SPM dependency for `TokenStore`.
- The app's push-registration and local-database services that implement
  `AuthLifecycleObserving`.
