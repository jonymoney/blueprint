# otp-email-template — Spec

<!-- Extracted from komorebi server/src/auth/email-templates.ts (2026-08-25).
     Owner decisions at the checkpoint: subject leads with the raw code
     (glanceable in notification banners); light AND dark palettes (the
     source is light-only — dark support was added by decision); the
     magic-link template stays out of the catalog entirely. -->

## Problem

OTP sign-in sends exactly one transactional email, and email clients are
hostile renderers: stylesheets stripped, remote assets blocked, dark modes
repainting colors unpredictably. This module is that email as a reusable
render unit — a fixed, battle-tested layout (centered card, brand header,
instruction, expiry line, large letter-spaced code) whose entire look
(light + dark palettes, fonts, copy, brand name) is supplied per project.
It generates as a single pure function the consuming auth module's OTP
sender calls.

## No-goals

- No magic-link (or any other) email template — OTP only, by owner decision.
- No sending — delivery is the consuming module's `EmailSender` concern.
- No images or remote assets of any kind (logo is typeset, not an `<img>`).
- No localization — copy strings are single-language parameters.
- No per-client hacks beyond the baseline (no MSO conditional comments); the
  light baseline IS the degradation path.
- No runtime theming — palettes are compiled in at generation time.

## Domain types

Generated into the consuming module's folder (no `core/` additions):

```ts
/** A fully rendered transactional email. */
export interface AuthEmail {
  subject: string;
  html: string;
  text: string;
}

/** Render the OTP sign-in email. Pure; no I/O. */
export function otpEmail(otp: string): AuthEmail;
```

## Module map

- **Now**: the OTP email render unit (one file, one exported function).
- **Next**: `auth-backend` — its email-OTP send hook calls `otpEmail` and
  posts the result through the core `EmailSender`.
- **Later**: further transactional templates on the same skeleton (welcome,
  security alert), localization.

## Contract

### Endpoints

None — a pure render function, no HTTP surface.

### Events

- **Emits**: none.
- **Listens**: none.

### Business rules

1. When any email renders, then the structure is: full-width background
   table → centered card (max 440px, surface color, 1px border, 12px
   radius) → brand name, instruction, expiry line, code — with the
   "didn't request this?" line outside the card.
2. When rendering HTML, then table layout (`role="presentation"`) with the
   light palette inlined on every element — the baseline every client can
   render — and no external assets.
3. When the client prefers dark, then an embedded `<style>` block swaps all
   seven color roles to the dark palette via
   `@media (prefers-color-scheme: dark)` class overrides (`!important`),
   with `color-scheme: light dark` + `supported-color-schemes: light dark`
   metas. Clients that strip `<style>` fall back to the light baseline —
   degraded, never broken.
4. When fonts are declared, then {FONT_DISPLAY} and {FONT_BODY} always carry
   full system fallback stacks — custom faces rarely load in email and the
   layout must hold on fallbacks.
5. When the code renders, then display font, 34px bold, 0.18em
   letter-spacing, accent color — the visual focus of the email.
6. When the subject is composed, then it leads with the raw code:
   `{otp} {SUBJECT_SUFFIX}` — readable from a notification banner without
   opening the email (owner-confirmed trade-off).
7. When any email is composed, then a plain-text part is always included:
   instruction, code, expiry, ignore line.
8. When the expiry copy renders, then it states the consuming module's
   actual OTP policy ({EXPIRY_COPY} must match auth-backend rule 6 — 5
   minutes by default).
9. When a project instantiates the template, then both palettes (7 roles ×
   light/dark), fonts, brand name, and all copy come from Parameters; only
   the skeleton is fixed.

### Examples

```ts
otpEmail('482913')
```

```json
{
  "subject": "482913 is your Jizo sign-in code",
  "text": "Your Jizo sign-in code is 482913.\n\nIt expires in 5 minutes. If you didn't request it, ignore this email.",
  "html": "<!doctype html>… (light styles inline; dark overrides in <style>; card ≤440px; code 34px/0.18em accent)"
}
```

### Error table

| Error code | HTTP status | When it occurs |
|---|---|---|
| — | — | Pure function; no failure modes |

## Env vars

None.

## Integration surface

- **`otpEmail(otp)`** — the consuming module's OTP send hook calls it and
  posts the result through the core `EmailSender`. The file generates into
  the consuming module's own folder (e.g.
  `src/modules/auth-backend/otp-email.ts`), so no cross-module import
  exists.
- **Copy coupling**: {EXPIRY_COPY} must state the consumer's real OTP
  expiry; changing the OTP policy means changing this parameter with it.

## Parameters

| Parameter | Meaning | Example |
|---|---|---|
| `BRAND_NAME` | Typeset brand header + subject/copy insertions | `Jizo` |
| `PALETTE_LIGHT` | bg, surface, text, muted, border, accent, accent-ink | `#FBF1E3, #FFF8EE, #2E1A12, #7A5848, #EAD6BE, #C2410C, #FFF8EE` |
| `PALETTE_DARK` | Same seven roles for dark-mode clients | `#221610, #2C1D14, #F4E8DA, #C7A38B, #4B3626, #ED6A2F, #FFF8EE` |
| `FONT_DISPLAY` | Display stack (brand header, code) with system fallbacks | `'Bricolage Grotesque', ui-sans-serif, system-ui, sans-serif` |
| `FONT_BODY` | Body stack with system fallbacks | `'Inter', ui-sans-serif, system-ui, sans-serif` |
| `INSTRUCTION_COPY` | Line above the code | `Enter this code to sign in.` |
| `EXPIRY_COPY` | Muted line stating the OTP policy | `The code expires in 5 minutes.` |
| `FOOTER_COPY` | Line outside the card | `Didn't request this? You can ignore it.` |
| `SUBJECT_SUFFIX` | Appended to the leading code | `is your Jizo sign-in code` |

## Outside the framework

- Delivery and deliverability (provider account, verified sending domain,
  SPF/DKIM) — owned by the consuming module's `EmailSender` setup.
- The consuming auth module's OTP policy — this template only *states* it.
