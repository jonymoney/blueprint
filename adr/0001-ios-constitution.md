# ADR-0001: Add an iOS constitution for client-side module specs

- **Status**: proposed
- **Date**: 2026-08-19

## Context

The catalog's CONSTITUTION.md is TypeScript/NestJS/Postgres-only. The first
client-side extractions (auth-ios, onboarding-ios, from komorebi's iOS app)
have no constitution to be generic against, and several SPEC_TEMPLATE.md
sections are server-shaped (Endpoints, Env vars, prisma models).

## Decision

Add `CONSTITUTION.ios.md` — immutable rules for iOS module specs, parallel to
the server constitution. iOS specs follow SPEC_TEMPLATE.md with a fixed
reinterpretation of the server-shaped sections:

- **Endpoints** → the backend API surface the module consumes (each row
  references the serving catalog module when one exists).
- **Events** → in-process events (NotificationCenter names, typed payloads).
- **Env vars** → build-time configuration (per-environment values, Info.plist
  keys, entitlements).

## Consequences

- Client modules become catalog citizens; a project can instantiate matched
  pairs (auth-backend + auth-ios) whose contracts reference each other.
- Two constitutions must not drift on shared contracts: where an iOS spec's
  Endpoints section names a server module, the server spec is the source of
  truth for those shapes.
- CONSTITUTION.md is unchanged. SPEC_TEMPLATE.md is unchanged (the
  reinterpretation lives here and in CONSTITUTION.ios.md).
- `blueprint <module>` for an iOS spec generates into an iOS project
  (Features/<Name>/ + declared Core/ additions) instead of src/modules/.
