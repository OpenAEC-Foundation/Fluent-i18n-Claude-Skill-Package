# Skill Index — Fluent i18n Claude Skill Package

16 deterministic skills for Project Fluent (Mozilla i18n) covering FTL syntax, @fluent/bundle API, @fluent/react integration, and @fluent/langneg language negotiation.

---

## fluent-core/ (3 skills)

| Skill | Description | Key Topics |
|-------|-------------|------------|
| [fluent-core-architecture](skills/source/fluent-core/fluent-core-architecture/SKILL.md) | Design philosophy, asymmetric localization, runtime model, Fluent vs ICU | FluentBundle, FluentResource, Message, Pattern |
| [fluent-core-bundle-api](skills/source/fluent-core/fluent-core-bundle-api/SKILL.md) | @fluent/bundle complete API: constructor, addResource, getMessage, formatPattern | FluentBundle, FluentResource, FluentType, error handling |
| [fluent-core-langneg](skills/source/fluent-core/fluent-core-langneg/SKILL.md) | Language negotiation with filtering/matching/lookup strategies | negotiateLanguages, acceptedLanguages, BCP 47 |

## fluent-syntax/ (5 skills)

| Skill | Description | Key Topics |
|-------|-------------|------------|
| [fluent-syntax-messages](skills/source/fluent-syntax/fluent-syntax-messages/SKILL.md) | FTL message syntax, multiline, placeables, whitespace rules, comments | Message, Pattern, Variable, Comment, indentation |
| [fluent-syntax-selectors](skills/source/fluent-syntax/fluent-syntax-selectors/SKILL.md) | Select expressions, CLDR plural categories, ordinal/cardinal, nested | SelectExpression, Variant, CLDR, NUMBER(type:"ordinal") |
| [fluent-syntax-terms](skills/source/fluent-syntax/fluent-syntax-terms/SKILL.md) | Terms, parameterized terms, term attributes, visibility rules | Term, TermReference, NamedArgument, -brand-name |
| [fluent-syntax-attributes](skills/source/fluent-syntax/fluent-syntax-attributes/SKILL.md) | Message/term attributes, value-less messages, compound messages | .attribute, getMessage().attributes, attrs prop |
| [fluent-syntax-functions](skills/source/fluent-syntax/fluent-syntax-functions/SKILL.md) | NUMBER/DATETIME builtins, custom functions, bidirectional text | NUMBER(), DATETIME(), FluentFunction, useIsolating |

## fluent-impl/ (4 skills)

| Skill | Description | Key Topics |
|-------|-------------|------------|
| [fluent-impl-react](skills/source/fluent-impl/fluent-impl-react/SKILL.md) | @fluent/react: Provider, Localized, useLocalization, DOM overlays, SSR | LocalizationProvider, Localized, ReactLocalization |
| [fluent-impl-locale-loading](skills/source/fluent-impl/fluent-impl-locale-loading/SKILL.md) | Async FTL loading, lazy generators, file organization, caching | fetch, FluentResource, CachedSyncIterable |
| [fluent-impl-locale-switching](skills/source/fluent-impl/fluent-impl-locale-switching/SKILL.md) | Dynamic locale switching, React state pattern, preference persistence | useState, negotiateLanguages, localStorage |
| [fluent-impl-project-setup](skills/source/fluent-impl/fluent-impl-project-setup/SKILL.md) | Complete project scaffolding, npm deps, directory structure, tooling | npm install, directory layout, @fluent/syntax |

## fluent-errors/ (2 skills)

| Skill | Description | Key Topics |
|-------|-------------|------------|
| [fluent-errors-parsing](skills/source/fluent-errors/fluent-errors-parsing/SKILL.md) | FTL parse errors, tab/space issues, junk recovery, strict validation | addResource errors, FluentParser, Junk, Annotation |
| [fluent-errors-resolution](skills/source/fluent-errors/fluent-errors-resolution/SKILL.md) | Runtime errors, 10 error types, throw vs collect modes, debugging | formatPattern errors, FluentNone, ReferenceError |

## fluent-agents/ (2 skills)

| Skill | Description | Key Topics |
|-------|-------------|------------|
| [fluent-agents-review](skills/source/fluent-agents/fluent-agents-review/SKILL.md) | Validation checklist, 25 anti-patterns, common AI mistakes | FTL syntax, API usage, React integration checks |
| [fluent-agents-project-scaffolder](skills/source/fluent-agents/fluent-agents-project-scaffolder/SKILL.md) | Generate complete Fluent project: React, vanilla JS, or Node.js | File templates, dependency list, decision trees |
