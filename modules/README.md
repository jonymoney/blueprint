# modules/

Catalog of generic, parameterized module specs. One file per module
(`<module>.md`), sub-specs as `<module>.<part>.md` when a generable unit
would exceed the 500-line output limit.

Every spec follows [SPEC_TEMPLATE.md](../SPEC_TEMPLATE.md) exactly. Specs
enter the catalog via `blueprint extract` (see [SKILL.md](../SKILL.md)).

Each spec has a human companion doc, `<module>.docs.md`: plain-language
summary, mermaid diagrams (rendered by GitHub), and screenshots in
`<module>.assets/` for modules with user-facing screens. Docs are for
reading only — generation uses the spec exclusively.
