# Approved Sources Registry

All skills in this package MUST be verified against these approved sources. No other sources are authoritative.

## Official Documentation

| Source | URL | Purpose |
|--------|-----|---------|
| Project Fluent Website | https://projectfluent.org | Primary reference for Fluent concepts and specification |
| Fluent Syntax Guide | https://projectfluent.org/fluent/guide/ | Complete FTL syntax tutorial and reference |
| FTL Specification | https://github.com/projectfluent/fluent/wiki/Spec:-Fluent-Syntax-1.0 | Formal FTL syntax specification |

## Source Code

| Source | URL | Purpose |
|--------|-----|---------|
| Project Fluent Organization | https://github.com/projectfluent | All official Fluent repositories |
| fluent.js Monorepo | https://github.com/projectfluent/fluent.js | JavaScript implementation (@fluent/bundle, @fluent/react, @fluent/langneg) |
| Fluent Specification Repo | https://github.com/projectfluent/fluent | FTL spec, EBNF grammar, test fixtures |

## Package Documentation

| Source | URL | Purpose |
|--------|-----|---------|
| @fluent/bundle (npm) | https://www.npmjs.com/package/@fluent/bundle | Core runtime: FluentBundle, FluentResource, formatPattern |
| @fluent/react (npm) | https://www.npmjs.com/package/@fluent/react | React bindings: LocalizationProvider, Localized, useLocalization |
| @fluent/langneg (npm) | https://www.npmjs.com/package/@fluent/langneg | Language negotiation: negotiateLanguages |
| @fluent/syntax (npm) | https://www.npmjs.com/package/@fluent/syntax | Parser/serializer for FTL files (tooling, not runtime) |

## API References

| Source | URL | Purpose |
|--------|-----|---------|
| @fluent/bundle API | https://github.com/projectfluent/fluent.js/tree/main/fluent-bundle | Bundle API source + README |
| @fluent/react API | https://github.com/projectfluent/fluent.js/tree/main/fluent-react | React integration source + README |
| @fluent/langneg API | https://github.com/projectfluent/fluent.js/tree/main/fluent-langneg | Language negotiation source + README |

## Community (for Anti-Pattern Research)

| Source | URL | Purpose |
|--------|-----|---------|
| fluent.js Issues | https://github.com/projectfluent/fluent.js/issues | Real-world error patterns, bug reports |
| Fluent Discourse | https://discourse.mozilla.org/c/fluent/11 | Mozilla community discussions |
| MDN Intl Reference | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl | Underlying Intl API that Fluent leverages |

## Claude / Anthropic (Skill Development Platform)

| Source | URL | Purpose |
|--------|-----|---------|
| Agent Skills Standard | https://agentskills.io | Open standard |
| Agent Skills Spec | https://github.com/agentskills/agentskills | Specification |
| Agent Skills Best Practices | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices | Authoring guide |

## OpenAEC Foundation

| Source | URL | Purpose |
|--------|-----|---------|
| ERPNext Skill Package | https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package | Methodology template |
| Tauri Skill Package | https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package | Proven 27-skill package |
| Blender Skill Package | (reference only) | Proven 73-skill package |

## Source Verification Rules

1. **Primary sources ONLY** — Official Fluent docs, official repos, official npm package docs.
2. **NEVER trust random blog posts** — Even popular ones may be outdated or use deprecated APIs.
3. **Verify FTL against the specification** — Every FTL snippet in a skill MUST match the formal spec.
4. **Note when source was last verified** — Track in the table below.
5. **Cross-reference if docs are sparse** — When official docs lack detail, verify against source code in the monorepo.

## Last Verified

| Technology | Date | Action | Notes |
|------------|------|--------|-------|
| Project Fluent | 2026-03-19 | Initial setup | All URLs pending verification |
