# blueprint

Spec-driven codegen framework for Claude Code. Specs are the source of truth;
code is a regenerable artifact, generated one-shot per module. This repo is
both the skill and the catalog of parameterized module specs.

- [SKILL.md](SKILL.md) — the skill (two verbs, full flow)
- [CONSTITUTION.md](CONSTITUTION.md) — immutable rules every module follows
- [SPEC_TEMPLATE.md](SPEC_TEMPLATE.md) — the rigid spec format
- [modules/](modules/) — the catalog
- [adr/](adr/) — decision records

## How to use it

### Install (once)

Symlink this repo into your personal skills so it's available in every
Claude Code session:

```bash
ln -sfn ~/Documents/Dev/blueprint ~/.claude/skills/blueprint
```

### Extract a module (existing code → catalog)

Open Claude Code in the project that has the code and say:

```
blueprint extract auth
```

or with a free-text brief:

```
blueprint extract "the auth in server/src, sessions only, call it auth-backend"
```

The skill analyzes the module, shows you a summary + mermaid diagrams, then
**stops at a human checkpoint**: you confirm or correct every business rule
(implicit ones are marked `[VERIFY]`). Only after your approval does it
parameterize and write the spec to this repo's `modules/<name>.md` — through
the symlink, straight into this working tree. Review and commit it here.

### Generate a module (catalog → project)

Open Claude Code in the consuming project and say:

```
blueprint auth-backend
```

The skill reads the constitution and the spec, asks you for every value in
the spec's Parameters table, instantiates the filled spec into the project's
`specs/`, and generates the module one-shot: `src/modules/<name>/` + e2e
tests + the env schema / `.env.example` updates the spec declares. It
verifies the "compiled" criteria (tsc + lint + tests green, nothing touched
outside the module) and always ends with a **handoff report**
(`specs/<name>.handoff.md`): env vars you must set, pending integration
steps, and curl examples to smoke-test.

### Change the rules

CONSTITUTION.md and published contracts only change with an ADR — copy
[adr/0000-template.md](adr/0000-template.md).
