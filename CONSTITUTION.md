# CONSTITUTION

Immutable rules for every blueprint module. Decided — do not re-litigate.
Changes to this file or any published contract require an ADR in `adr/`.

## Stack

- TypeScript strict mode. NestJS.
- Postgres + Prisma. `schema.prisma` is a readable contract. Migrations are
  manual and never regenerated.
- zod for DTO validation: schemas as values, types via `z.infer`. Validation
  runs through a global pipe defined in `core/lib`.
- vitest + supertest for tests.
- No hard cap on external deps, but every dep must be justified in the spec
  that introduces it.

## Structure

- `src/modules/<name>/` — regenerable units. Each is generated one-shot from
  its spec and may be thrown away and regenerated at any time.
- `src/core/` — shared code (domain types, prisma, lib). Never regenerated
  one-shot. Edited deliberately, by hand or with an ADR-backed change.

## Module rules

- A module fits in ≤500 lines of generated output. If a spec can't be
  generated within that, split it into sub-specs.
- Modules import only from `core/` and `node_modules`. Never from another
  module.
- Controllers hold no logic — parse, delegate, respond.
- Services are injected via token/interface, never referenced by concrete
  class across boundaries.

## Cross-module communication

- Synchronous queries: interfaces declared in `core/`, implemented by the
  owning module, injected by token.
- Side effects: typed events via `@nestjs/event-emitter`.
- Never request/response over events.

## API

- All responses use the standard `{ data, error }` envelope defined in
  `core/lib`.

## Env handling

- `core/lib/env.ts` holds a single zod schema validating all env vars at
  startup. Fail fast.
- Modules never read `process.env` directly — they consume typed config
  injected from core.
- Every generated module updates the env schema and `.env.example` with the
  vars declared in its spec.

## Providers

- External integrations are never imported directly by modules. Each lives in
  `core/lib` behind an interface:
  - Email: Resend behind `EmailSender`
  - SMS: Surge behind `SmsSender`
- Modules inject the interface; tests mock it.

## Deploys

- Target: Railway, Postgres as an attached service, config via env vars.
- Deploy config (`railway.toml`, env vars) lives outside the framework and is
  never regenerated.

## Regeneration

- Regenerate > patch: if a change touches >30% of a module, regenerate it
  from spec.
- "Compiled" means: `tsc` + lint + module tests green, and nothing was
  touched outside the module's folder + its e2e spec (+ the env schema /
  `.env.example` updates its spec declares).
