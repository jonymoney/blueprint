# onboarding-ios — Spec

<!-- Extracted from komorebi ios/ (2026-08-19), on CONSTITUTION.ios.md
     (ADR-0001). The source app's onboarding is a placeholder screen; this
     spec generalizes it to a parameterized screen sequence over the same
     gate + coordinator contract. Companion spec: modules/auth-ios.md. -->

## Problem

A first-launch app needs a short introduction before asking anyone to sign
in: a sequence of static screens shown exactly once, a persisted completion
flag, and a routing contract the app shell can gate on. This module owns
that gate and the screen scaffold; the screens' content is entirely
parameterized.

## No-goals

- No authentication — sign-in is `modules/auth-ios.md`, mounted after this
  gate completes.
- No permission priming (notifications, location) — those prompts belong to
  the features that need them.
- No server-driven or remote-configured content; screens are compiled in.
- No A/B testing or analytics instrumentation.
- No re-showing after sign-out — the flag is per-install, not per-session.

## Domain types

Added to `Core/` (namespaced setting):

```swift
extension Settings {
    final class Onboarding {
        @Setting("onboarding", "has_completed", defaultValue: false)
        var hasCompleted: Bool
    }
}
```

## Module map

- **Now**: onboarding feature (coordinator + view controller scaffold
  rendering {SCREENS}), the persisted completion flag.
- **Next**: `auth-ios` — the routing gate hands off to sign-in.
- **Later**: permission-priming pages, remote-configurable copy, a
  re-runnable "what's new" variant keyed by version.

## Contract

### Endpoints

None — fully offline.

### Events

None — completion is reported through the coordinator callback.

### Business rules

1. When the app launches and `hasCompleted` is false, then onboarding is the
   root screen — before sign-in and before anything else.
2. `[VERIFY]` When onboarding precedes sign-in, that ordering is the rule
   (intro shown to signed-out first-run users, not after first sign-in).
3. When the user finishes the last screen, then set `hasCompleted = true`
   (persisted) and fire `onFinished`; the app shell routes onward (sign-in
   when unauthenticated).
4. When `hasCompleted` is true, then onboarding never shows again for this
   install — including after sign-out or session expiry.
5. When a screen renders, then all colors and fonts come from `Theme`
   tokens and content scales via the readable content guide (Catalyst).
6. When {SCREENS} has multiple entries, then they page horizontally with a
   page indicator; the final page's button label is
   {FINAL_BUTTON_LABEL}, earlier pages' {CONTINUE_BUTTON_LABEL}.

### Examples

Two-screen instantiation, first launch:

```
launch  hasCompleted=false        → onboarding shown as root
page 1  "Leave something behind"  → Continue
page 2  "Earn park stamps"        → Get started (final)
tap     hasCompleted=true persisted → onFinished → sign-in screen
relaunch hasCompleted=true        → onboarding skipped
```

### Error table

| Error | Surfaced as | When it occurs |
|---|---|---|
| — | — | No failure modes: no network, no input; the setting write is the platform's |

## Env vars

None — no build-time configuration.

## Integration surface

- **`OnboardingCoordinator`** — `start() -> UIViewController` + `onFinished`
  callback. The app coordinator mounts it as root when
  `settings.onboarding.hasCompleted` is false, sets the flag true in
  `onFinished`, and routes onward (see auth-ios rule 1).
- **`Settings.Onboarding.hasCompleted`** — the gate flag; the app
  coordinator reads it at launch. Nothing else writes it.

## Parameters

| Parameter | Meaning | Example |
|---|---|---|
| `SCREENS` | Ordered screen content: title, body copy, SF Symbol or asset name per page | `[{title:"Leave something behind", body:"…", symbol:"mappin.and.ellipse"}, …]` |
| `CONTINUE_BUTTON_LABEL` | Button label on non-final pages | `Continue` |
| `FINAL_BUTTON_LABEL` | Button label on the last page | `Get started` |

## Outside the framework

- The app shell's routing (onboarding → sign-in → main) — this module only
  provides the gate flag and coordinator; the shell owns the order.
- The `@Setting` property wrapper, `Container` DI, and `Theme` tokens (ios
  constitution core).
- Any artwork the {SCREENS} reference beyond SF Symbols (asset catalog
  entries are the consuming project's).
