# CONSTITUTION — iOS

Immutable rules for every iOS blueprint module. Decided — do not re-litigate.
Changes to this file or any published contract require an ADR in `adr/`.
Adopted by ADR-0001. The server CONSTITUTION.md governs backend modules; where
an iOS spec consumes a backend catalog module, the server spec is the source
of truth for the API shapes.

## Stack

- Swift, strict concurrency. UIKit, programmatic UI only — no storyboards,
  no XIBs.
- SnapKit for all Auto Layout. Never raw NSLayoutConstraint or frames.
- GRDB (SQLite) for local storage.
- Swift Package Manager for dependencies; every dep justified in the spec
  that introduces it. XcodeGen (`project.yml`) generates the project.
- XCTest for tests; protocol mocks, no mocking frameworks.

## Structure

- `Features/<Name>/` — regenerable units: coordinator + view model + view
  controller together, organized by feature, not by layer. Generated one-shot
  from a spec; may be thrown away and regenerated.
- `Core/` and `Models/` — shared code (services, networking, theme, domain
  types). Never regenerated one-shot; edited deliberately. A spec may declare
  the `Core/` types it adds, exactly as they will appear.

## Module rules

- A module fits in ≤500 lines of generated output per generable unit.
- MVVM-C: coordinators own navigation and expose `start() -> UIViewController`
  plus completion callbacks; view models own state; view controllers render.
- View models expose state as a single `State` enum via a Combine
  `Published` publisher, behind a protocol.
- Protocol-first DI: every service is a protocol with at least the real
  implementation and a test mock; resolved via the app's `Container` at
  bootstrap (`@Inject`) or constructor injection. No singletons.
- Controllers/views hold no business logic — bind, render, forward.
- `final class` by default; `[weak self]` in escaping closures; no force
  unwraps in production code.

## UI

- All colors and fonts through `Theme.Color` / `Theme.Font` tokens (asset
  catalog, light + dark from day one). SF Symbols for iconography.
- Respect the readable content guide and size classes (Mac Catalyst enabled);
  no hardcoded widths.
- Animations subtle and functional.

## Networking

- Typed endpoints (`Endpoint<Response>`) sent through the shared `APIClient`;
  interceptors handle cross-cutting concerns (auth headers, session expiry).
- Modules never construct URLs or headers inline.

## Secrets & settings

- Session tokens and secrets live in the Keychain, never UserDefaults.
- User preferences via the namespaced `@Setting` property wrapper.

## Events

- In-process cross-feature signals via `Notification.Name` constants declared
  in the owning module; payloads typed, minimal, documented in the spec.

## Regeneration

- Regenerate > patch: a change touching >30% of a module means regenerate
  from spec.
- "Compiled" means: the app builds, lint passes, module tests green, and
  nothing was touched outside the module's feature folder + the `Core/`
  additions its spec declares.
