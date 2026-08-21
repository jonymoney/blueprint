# footer-web — Spec

<!-- Constitution note: the catalog has no web-frontend constitution
     (CONSTITUTION.md is NestJS backend; CONSTITUTION.ios.md is iOS).
     This spec carries its own minimal ground rules: a single React client
     component, styled via the host site's CSS custom-property tokens and
     utility classes, zero runtime dependencies beyond React, no data
     fetching. If a CONSTITUTION.web.md lands later, reconcile via ADR. -->

## Problem

A site footer that signs off with warmth instead of boilerplate: "Built with
♥ from `<place>`", cycling through the places the author has been. Naive
implementations jump — swapping "Bend" for "Puerto Escondido" reflows the
line and the spacing stutters. This module makes the place name's slot
animate its width to each name's real rendered width, so the sentence keeps a
normal single space around the name and glides between names, with each
incoming name fading in. A secondary copyright line sits below.

## No-goals

- No data fetching or CMS: the places list is a build-time constant supplied
  as a parameter, not pulled from an API.
- No screen-reader announcements of place changes: the ticker is decorative;
  no `aria-live` (confirmed by owner).
- No pause/hover/manual controls: it cycles on a fixed interval, always.
- No SSR width computation: the slot is `auto` width until the first
  client-side measurement lands.

## Domain types

The module owns no shared domain types. Its entire surface is one component
with internal types:

```ts
// places: ordered list; the ticker walks it top to bottom and wraps.
// Order carries no semantics — any ordered list is valid.
type Place = string;

export default function Footer(): JSX.Element; // no props
```

## Module map

- **Now**: the `Footer` component + its `city-in` keyframes (global CSS).
- **Next**: none expected — leaf module.
- **Later**: optional variant that reads places from the host's data layer
  (e.g. the same source as a travel map), if a consuming site has one.

## Contract

### Endpoints

None. This is a client-side presentational component; it makes no requests.

### Events

- **Emits**: none.
- **Listens**: none (DOM timer only, internal).

### Business rules

All confirmed by owner 2026-08-21.

- When `TICK_MS` elapses, then advance to the next place, wrapping to the
  first after the last.
- When the place changes, then the slot's width transitions over 300 ms
  ease-out to the incoming name's natural rendered width, measured
  synchronously before paint (hidden measurer span with identical font
  styles; React: `useLayoutEffect` reading `offsetWidth`).
- When a new place enters, then its text node is recreated (React:
  `key={place}`) so the entry animation replays: fade from 0 + rise 6 px,
  0.4 s ease.
- When the user prefers reduced motion, then the entry fade/rise is disabled
  but the width transition still runs (deliberate: it is a subtle
  layout-level glide, not a decorative flourish).
- When the component first renders, then the slot has no explicit width
  (`auto`) until the first measurement lands.
- When the component unmounts, then the interval is cleared.

### Examples

Rendered DOM (place = "Mexico City", lead = "Built with ♥ from",
secondary = "© 2026 Jony Money"):

```html
<footer class="relative mt-16 text-center text-sm text-gray-600 dark:text-gray-400">
  <p>
    Built with <span class="text-accent">♥</span> from
    <span class="inline-block overflow-hidden whitespace-nowrap align-bottom
                 font-medium text-foreground transition-[width] duration-300
                 ease-out" style="width: 87px">
      <span class="city-in inline-block">Mexico City</span>
    </span>
  </p>
  <span aria-hidden="true"
        class="invisible absolute whitespace-nowrap font-medium">Mexico City</span>
  <p class="mt-2 text-xs">© 2026 Jony Money</p>
</footer>
```

Global CSS the component requires:

```css
@keyframes city-in {
  from { opacity: 0; transform: translateY(6px); }
  to   { opacity: 1; transform: none; }
}
@media (prefers-reduced-motion: no-preference) {
  .city-in { animation: city-in 0.4s ease; }
}
```

### Error table

| Error code | HTTP status | When it occurs |
|---|---|---|
| — | — | No error surface: no I/O, no user input. An empty `PLACES` list must be rejected at build/review time (render guard not required). |

## Env vars

None.

| Name | Purpose | Example | Required |
|---|---|---|---|
| — | — | — | — |

## Integration surface

- **Component**: render `<Footer />` once, at the end of the page body. No
  props; all variation comes from Parameters at instantiation.
- **Global CSS**: the `city-in` keyframes block above must be added to the
  host's global stylesheet.
- **Design tokens consumed**: `--accent` (the ♥), foreground color (the place
  name), a muted gray for the sentence. Map them to the host's tokens at
  instantiation; the component references them via the host's utility classes
  or plain CSS.
- **Runtime**: must be a client component ("use client" in Next.js App
  Router) — it uses state, a timer, and DOM measurement.

## Parameters

| Parameter | Meaning | Example |
|---|---|---|
| `PLACES` | Ordered list of place names the ticker cycles through (wraps; order is display order, no other semantics) | `["Mexico City", "Oaxaca", "Puerto Escondido", "Coyoacán", "San Francisco", "Seattle", "Portland", "Bend", "Salt Lake City", "Yosemite", "Big Sur", "Zion", "Tokyo", "Kyoto", "Nagano", "Furano"]` |
| `LEAD_TEXT` | Sentence before the place; one glyph may be accent-colored | `Built with ♥ from` (♥ accented) |
| `SECONDARY_LINE` | Small muted line below the sentence | `© 2026 Jony Money` |
| `TICK_MS` | Interval between place changes | `2500` |
| `ACCENT_TOKEN` | Host CSS token/class for the accent glyph | `--accent` → `#8e4e75` light / `#c9799f` dark |
| `NAME_STYLE` | Weight/color treatment of the place name | `font-medium`, foreground color |

## Outside the framework

- The host site's design tokens (accent, foreground, muted grays) and its
  styling system (Tailwind or plain CSS — the spec's classes translate 1:1).
- A React 18+ client runtime; in Next.js App Router the file needs
  `"use client"`.
- Curation of the places list over time — editing the constant is a content
  change, not a regeneration.
- No web constitution exists in this catalog yet (see note at top).
