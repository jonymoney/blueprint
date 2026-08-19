# auth-ios — Human docs

> Companion to [auth-ios.md](auth-ios.md). Reading only — generation uses
> the spec exclusively. Client half of [auth-backend](auth-backend.docs.md).

## What it does, in plain language

The sign-in screen and everything a session means on the device. One screen
with an Email/Phone toggle: enter your email or US phone number, receive a
6-digit code, type it (iOS autofills it from Messages/Mail), and you're in.
The returned Bearer token is stored in the Keychain and attached to every
API request from then on.

There is no refresh token — the token *is* the session. If any ordinary API
call comes back 401, the session is dead (expired or revoked server-side):
the app silently signs out, wipes everything the account left on the device
(database rows, cached images), and drops you back at sign-in. Errors on the
auth endpoints themselves (a wrong code) are just form errors — they never
end the session.

Sign-out does things in a deliberate order: first tell the server to stop
pushing to this device (while the session can still authorize that), then
revoke the session (best-effort), then always clear the token and wipe local
account data — so the next person to sign in inherits nothing.

The app plugs into session boundaries via `AuthLifecycleObserving`
(didSignIn / willSignOut / didSignOut) instead of the auth module knowing
about push or the database directly.

## Screens

The extracted app, as built (its email path still shows magic-link copy —
the spec replaces it with the same code-entry step the phone path uses):

| Sign-in — email | Sign-in — phone |
|---|---|
| ![Sign-in, email method](auth-ios.assets/sign-in-email.png) | ![Sign-in, phone method](auth-ios.assets/sign-in-phone.png) |

## Dependency map

```mermaid
flowchart LR
  subgraph feature["Features/Auth"]
    C["AuthCoordinator"]
    VC["AuthViewController"]
    VM["AuthViewModel<br/>State: idle→loading→codeSent→signedIn|error"]
  end
  subgraph corelib["Core/"]
    AM["AuthManager (AuthManaging)"]
    TS["TokenStore (Keychain)"]
    I["AuthInterceptor"]
    API["AuthAPI typed endpoints"]
  end
  APP["AppCoordinator"]
  OBS["AuthLifecycleObserving<br/>(push, DB wipe — app-registered)"]
  BE[("auth-backend<br/>/api/auth/*")]

  APP -- "start()/onFinished" --> C
  C --> VC --> VM --> AM
  AM --> API --> BE
  AM --> TS
  AM -. "didSignIn / willSignOut / didSignOut" .-> OBS
  I --> TS
  I -- "Bearer header on every request" --> BE
  I -- ".sessionExpired" --> APP
```

## Sign-in flow (email and phone are symmetric)

```mermaid
sequenceDiagram
  actor U as User
  participant VC as Sign-in screen
  participant AM as AuthManager
  participant BE as auth-backend

  U->>VC: enter email / US phone
  VC->>VC: phone: normalize to E.164 or inline error (no network)
  VC->>AM: request code
  AM->>BE: POST send-otp (email-otp / phone-number)
  BE-->>U: 6-digit code by email / SMS
  VC->>VC: code step (.oneTimeCode autofill, first responder)
  U->>VC: enter code
  AM->>BE: POST verify (sign-in/email-otp / phone-number/verify)
  alt wrong / expired code
    BE-->>VC: 400 → error alert, step restored, retry
  else not allowlisted
    BE-->>VC: 403 "Sign-ups are invite-only"
  else success
    BE-->>AM: 200 {token}
    AM->>AM: Keychain store, didSignIn() observers (fire-and-forget)
    VC-->>U: → main app
  end
```

## Session lifecycle

```mermaid
sequenceDiagram
  participant Any as Any API call
  participant I as AuthInterceptor
  participant APP as AppCoordinator
  participant AM as AuthManager

  Note over Any,I: every request: attach Bearer token if present
  Any-->>I: 401 (path NOT /api/auth/*)
  I->>APP: post .sessionExpired
  APP->>AM: signOut()
  Note over AM: willSignOut (push unregister)<br/>→ revoke server session (best-effort)<br/>→ clear Keychain (always)<br/>→ didSignOut (wipe DB + image cache)
  APP->>APP: evict to sign-in
```

Launch is optimistic: a token in the Keychain routes straight to the main
app without validation — a stale token is evicted by the first 401.

## What you must set up (digest)

- **Build config**: per-environment API base URL; a Keychain service id.
- **Parameters**: phone region normalizer (must match auth-backend's
  validator), screen copy, legacy Keychain keys to clean up.
- **To consume it**: mount `AuthCoordinator` when unauthenticated, subscribe
  to `.sessionExpired`, install `AuthInterceptor` on the shared `APIClient`,
  and register `AuthLifecycleObserving` observers for push registration and
  account-data wipe. Full details: spec's *Integration surface*.
- **Routing order**: the [onboarding-ios](onboarding-ios.docs.md) gate runs
  first on a fresh install; sign-in second; main app third.
