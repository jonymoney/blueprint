# <Module name> — Spec

<!-- Rigid format. Every section below is required, in this order.
     A spec missing a section is invalid. -->

## Problem

What this module solves, in a paragraph. No solutioning here.

## No-goals

Minimum 3. What this module deliberately does not do.

- …
- …
- …

## Domain types

The types this module owns or extends in `core/`. TypeScript, exactly as they
will appear.

```ts
```

## Module map

- **Now**: what this spec generates.
- **Next**: the adjacent module(s) expected to follow.
- **Later**: known but deferred.

## Contract

### Endpoints

For each endpoint: method + path, input DTO (zod schema), output DTO. All
responses wrapped in the `{ data, error }` envelope.

### Events

- **Emits**: event name → typed payload.
- **Listens**: event name → typed payload → what it does.

### Business rules

Every rule as "when X then Y". Rules extracted from code but not confirmed by
a human are marked `[VERIFY]`.

### Examples

Concrete request → response pairs, at least one per endpoint. Real JSON.

### Error table

| Error code | HTTP status | When it occurs |
|---|---|---|

## Env vars

Every env var the module requires.

| Name | Purpose | Example | Required |
|---|---|---|---|

## Integration surface

Everything the module exposes for consumers. For each item: what a consuming
module must do to use it.

- **Guards/decorators**: e.g. `@Authenticated` — apply to routes needing auth.
- **core/ interfaces implemented**: interface → token → how to inject.
- **Events emitted**: event → when to subscribe and why.

## Parameters

Project-specific values filled at instantiation time. Everything else in this
spec is generic.

| Parameter | Meaning | Example |
|---|---|---|

## Outside the framework

What this module assumes exists but does not manage (deploy config,
migrations, external accounts, DNS, …).
