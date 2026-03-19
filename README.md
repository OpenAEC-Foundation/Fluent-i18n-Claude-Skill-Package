# Fluent i18n Claude Skill Package

<p align="center">
  <strong style="font-size: 1.5em;">Deterministic Claude AI Skills for Project Fluent (Mozilla i18n)</strong>
</p>

![Claude Code Ready](https://img.shields.io/badge/Claude_Code-Ready-blue?style=flat-square&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZD0iTTEyIDJDNi40OCAyIDIgNi40OCAyIDEyczQuNDggMTAgMTAgMTAgMTAtNC40OCAxMC0xMFMxNy41MiAyIDEyIDJ6IiBmaWxsPSIjZmZmIi8+PC9zdmc+)
![Project Fluent](https://img.shields.io/badge/Project_Fluent-FTL-FF7139?style=flat-square)
![FTL + TypeScript](https://img.shields.io/badge/FTL_+_TypeScript-Dual_Coverage-FF7139?style=flat-square)
![Phase](https://img.shields.io/badge/Phase-1%20of%207-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)

**Deterministic Claude AI skills for Project Fluent localization — FTL syntax + TypeScript integration coverage.**

Built on the [Agent Skills](https://agentskills.org) open standard.

---

## Why This Exists

Without skills, Claude confuses Fluent FTL with ICU MessageFormat:

```
# Wrong — ICU MessageFormat syntax, NOT Fluent
welcome = Welcome, {username}! You have {count, plural, one {# message} other {# messages}}.
```

With this skill package, Claude produces correct Fluent FTL:

```ftl
# Correct — Fluent FTL syntax with proper select expression
welcome = Welcome, { $username }! You have { $count ->
    [one] { $count } message
   *[other] { $count } messages
}.
```

---

## Current Progress

**Phase 1 of 7** — Setup + Raw Masterplan (50%)

This package is under active development. Skills will be created after deep research into the Fluent specification and ecosystem.

## Planned Skill Categories

| Category | Purpose |
|----------|---------|
| **fluent-core/** | Architecture, bundle API, language negotiation |
| **fluent-syntax/** | FTL messages, selectors, terms, attributes, placeables |
| **fluent-impl/** | React integration, bundle setup, locale management |
| **fluent-errors/** | Parse errors, resolution failures, debugging |
| **fluent-agents/** | Validation, FTL generation orchestration |

## Technology Coverage

| Area | Topics |
|------|--------|
| **FTL Syntax** | Messages, placeables, selectors, terms, attributes, comments |
| **@fluent/bundle** | FluentBundle, FluentResource, formatPattern, getMessage |
| **@fluent/react** | LocalizationProvider, Localized, useLocalization |
| **@fluent/langneg** | negotiateLanguages, locale matching, fallback chains |
| **Patterns** | Plurals, number/date formatting, message references, parameterized terms |

## Version Compatibility

| Package | Versions | Notes |
|---------|----------|-------|
| @fluent/bundle | **0.18+** | Core runtime |
| @fluent/react | **0.15+** | React bindings |
| @fluent/langneg | latest | Language negotiation |
| @fluent/syntax | latest | Parser/serializer (tooling) |

## Installation

### Claude Code

```bash
# Option 1: Clone the full package
git clone https://github.com/OpenAEC-Foundation/Fluent-i18n-Claude-Skill-Package.git
cp -r Fluent-i18n-Claude-Skill-Package/skills/source/ ~/.claude/skills/fluent/

# Option 2: Add as git submodule
git submodule add https://github.com/OpenAEC-Foundation/Fluent-i18n-Claude-Skill-Package.git .claude/skills/fluent
```

### Claude.ai (Web)

Upload individual SKILL.md files as project knowledge.

## Quick Start

After installation, skills activate automatically based on context:

- **Write FTL messages** — ask Claude to create localization files → activates `fluent-syntax-messages`
- **Add plurals** — handle plural forms → activates `fluent-syntax-selectors`
- **Set up React i18n** — integrate Fluent with React → activates `fluent-impl-react`
- **Debug parse errors** — paste an FTL error → activates `fluent-errors-parsing`
- **Manage locales** — set up language switching → activates `fluent-impl-locale-management`

## Methodology

This package is developed using the **7-phase research-first methodology**, proven across multiple skill packages:

1. **Setup + Raw Masterplan** — Project structure and governance files
2. **Deep Research** — Comprehensive analysis of Fluent specification, source code, and community resources
3. **Masterplan Refinement** — Skill inventory refinement based on research findings
4. **Topic-Specific Research** — Deep-dive per skill topic
5. **Skill Creation** — Deterministic skill files following Agent Skills standard
6. **Validation** — Correctness, completeness, and consistency checks
7. **Publication** — GitHub release and documentation

## Documentation

| Document | Purpose |
|----------|---------|
| [ROADMAP.md](ROADMAP.md) | Project status (single source of truth) |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Quality guarantees and per-area requirements |
| [DECISIONS.md](DECISIONS.md) | Architectural decisions with rationale |
| [SOURCES.md](SOURCES.md) | Official reference URLs and verification rules |
| [WAY_OF_WORK.md](WAY_OF_WORK.md) | 7-phase development methodology |
| [LESSONS.md](LESSONS.md) | Lessons learned during development |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

## Related Projects

| Project | Description |
|---------|-------------|
| [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package) | 28 skills for ERPNext/Frappe development |
| [Blender-Bonsai Skill Package](https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package) | 73 skills for Blender, Bonsai, IfcOpenShell & Sverchok |
| [Tauri Skill Package](https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package) | 27 skills for Tauri 2 desktop development |
| [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) | Parent organization |

## License

[MIT](LICENSE)

---

Part of the [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) ecosystem.
