---
name: blueprint
description: >
  Spec-driven codegen framework. Specs are the source of truth; code is a
  regenerable artifact, generated one-shot per module. Use when the user says
  "blueprint <module>" (instantiate a catalog module into a project) or
  "blueprint extract <module>" (turn existing code into a catalog spec).
---

# blueprint

Specs are the source of truth. Code is a regenerable artifact. This repo is a
catalog of parameterized module specs (`modules/`) governed by
[CONSTITUTION.md](CONSTITUTION.md) and shaped by
[SPEC_TEMPLATE.md](SPEC_TEMPLATE.md).

Two verbs. Everything else is out of scope.

## How this runs

This repo is both the skill and the catalog. It is installed once as a
personal skill via symlink:

```
~/.claude/skills/blueprint -> ~/Documents/Dev/blueprint
```

So from **any** project, saying `blueprint extract <module>` or
`blueprint <module>` in Claude Code triggers this skill. The catalog path is
this skill's own directory (resolve the symlink):

- `blueprint extract` runs **in the target project** and writes the resulting
  spec to `<skill dir>/modules/<module>.md` — i.e. directly into this repo's
  working tree. Committing the new spec here is a manual step afterward.
- `blueprint <module>` runs **in the consuming project**, reads
  `<skill dir>/CONSTITUTION.md` and `<skill dir>/modules/<module>.md`, and
  writes only into the consuming project (specs/, src/modules/, env files).
  It never writes to this repo.

---

## Verb 1: `blueprint extract <module | "prompt">` — existing code → catalog

Turn a module in a real project into a generic, parameterized spec.

The argument is either a bare module name (`blueprint extract auth`) or a
quoted free-text brief (`blueprint extract "the auth flow in komorebi's
server — sessions only, skip the OAuth providers"`). A brief may name the
source project/path, scope what's in and out, and rename the resulting
module. Before analyzing:

0. **Resolve the brief.** Restate it as: source path + module boundary +
   exclusions + catalog name (`modules/<name>.md`). If the source project or
   boundary is ambiguous, ask — don't guess. Exclusions from the brief go
   into the spec's No-goals.

1. **Analyze.** Read the module in the target project end to end: endpoints,
   services, events, guards, env var reads, prisma models it touches, every
   import crossing its boundary.

2. **Human-comprehension pass.** Produce, in chat AND saved to
   `modules/<module>.docs.md` (companion doc for humans only — never an input
   to generation; the spec stays the single source of truth):
   - A plain-language summary of what the module does.
   - Mermaid diagrams: one sequence diagram per endpoint (request flow through
     controller → service → prisma/providers/events), plus one dependency map
     (what it imports from core/, which interfaces it implements, which events
     it emits/listens to).
   - The module's env vars and its integration surface (guards/decorators,
     core/ interfaces, events).
   - If the module has user-facing screens (e.g. onboarding), screenshots of
     each screen in `modules/<module>.assets/`, embedded in the docs file.

3. **HUMAN CHECKPOINT — before any parameterization.** Present every business
   rule as "when X then Y". Rules that are implicit in the code (not obviously
   intentional) are marked `[VERIFY]`. The human confirms or corrects each.
   **Do not proceed past this step without explicit approval.**

4. **Parameterize.** Move project-specific values into the spec's
   `## Parameters` section (e.g. for auth: identity provider, User fields,
   token expiry). What remains must be generic — valid for any project on
   this constitution.

5. **Write the spec.** `modules/<module>.md`, following SPEC_TEMPLATE.md
   exactly. If any single generable unit would exceed the 500-line output
   limit, split into sub-specs (`modules/<module>.<part>.md`), each
   independently generable and each ≤500 lines of output.

## Verb 2: `blueprint <module>` — catalog → project

Instantiate a catalog spec into the current project.

1. **Read** CONSTITUTION.md and `modules/<module>.md` in full.

2. **Ask the human for the Parameters values.** Every row of the spec's
   `## Parameters` table. Do not guess or default them.

3. **Instantiate.** Write the filled spec (parameters substituted, no open
   placeholders) to the project's `specs/<module>.md`.

4. **Generate one-shot.** Output exactly:
   - `src/modules/<module>/` — the module folder.
   - The module's e2e test spec.
   - The `core/lib/env.ts` schema additions and `.env.example` entries the
     spec's `## Env vars` section declares.
   Nothing else. No edits to core/ beyond the env schema, no edits to other
   modules.

5. **Verify "compiled".** `tsc` + lint + module tests green, and nothing
   touched outside the allowed set above. Fix within the module until green.

6. **HANDOFF REPORT — generation never ends with "done".** Write the report
   to `specs/<module>.handoff.md` AND present it in chat. It contains:

   - **Env vars**: which vars were added to `core/lib/env.ts` and
     `.env.example`, and which the human must still set — locally and in
     Railway.
   - **Integration steps**: from the spec's Integration surface, everything
     this generation did NOT implement in consumer code and remains pending.
     Each item is an actionable instruction for the next AI agent or human,
     e.g. "apply `@Authenticated` to endpoints that need protection",
     "subscribe to `user.registered` for welcome emails".
   - **Verification**: how to smoke-test the module end-to-end — curl
     examples taken from the spec's request→response examples.

---

## Ground rules (both verbs)

- The constitution is not negotiable at generation time. If a spec conflicts
  with it, the spec is wrong — stop and report.
- Regenerate > patch: a change touching >30% of a module means regenerate
  from spec.
- Changing CONSTITUTION.md or a published contract requires an ADR in `adr/`
  (start from `adr/0000-template.md`).
