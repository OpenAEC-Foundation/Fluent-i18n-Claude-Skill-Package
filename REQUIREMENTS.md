# REQUIREMENTS

## What This Skill Package Must Achieve

### Primary Goal
Enable Claude to write correct, spec-compliant FTL localization files and TypeScript integration code for Project Fluent — without hallucinating syntax or API patterns.

### What Claude Should Do After Loading Skills
1. Recognize Fluent/i18n/localization context from user requests
2. Select the correct skill(s) automatically based on the request
3. Write correct FTL syntax for messages, selectors, terms, and attributes
4. Generate proper TypeScript integration code using @fluent/bundle and @fluent/react
5. Avoid known anti-patterns and common AI mistakes with Fluent
6. Follow best practices documented in the skill references

### Quality Guarantees
| Guarantee | Description |
|-----------|-------------|
| Spec-correct | FTL syntax MUST conform to the Fluent specification |
| API-accurate | All @fluent/bundle and @fluent/react method signatures verified against official docs |
| Dual-language | Integration skills show both FTL and TypeScript |
| Anti-pattern-free | Known mistakes are explicitly documented and avoided |
| Deterministic | Skills use ALWAYS/NEVER language, not suggestions |
| Self-contained | Each skill works independently without requiring other skills |

---

## Per-Area Requirements

### 1. FTL Syntax — Messages & Patterns
| Requirement | Detail |
|-------------|--------|
| Simple messages | `key = Value` pattern, multiline text blocks |
| Placeables | Variable references `{ $variable }`, term references `{ -brand-name }` |
| Text elements | Proper whitespace handling, blank lines, indentation rules |
| Comments | `#` message comments, `##` group comments, `###` resource comments |
| Critical | Whitespace sensitivity — FTL is indentation-aware |

### 2. FTL Syntax — Selectors & Plurals
| Requirement | Detail |
|-------------|--------|
| Select expressions | `{ $count -> [one] ... *[other] ... }` pattern |
| CLDR plural categories | zero, one, two, few, many, other |
| Number formatting | `{ NUMBER($amount, style: "currency", currency: "EUR") }` |
| Date formatting | `{ DATETIME($date, dateStyle: "long") }` |
| Nested selectors | Multi-level select expressions |
| Critical | Default variant `*[other]` is MANDATORY in every select expression |

### 3. FTL Syntax — Terms & Attributes
| Requirement | Detail |
|-------------|--------|
| Terms | `-term-name = Value` pattern, term references in messages |
| Attributes | `.attribute = Value` pattern on messages and terms |
| Term parameterization | Passing arguments to term references |
| Critical | Terms NEVER appear in API output — they are internal-only references |

### 4. @fluent/bundle API
| Requirement | Detail |
|-------------|--------|
| FluentBundle | Constructor, `addResource()`, `getMessage()`, `formatPattern()` |
| FluentResource | Parsing FTL strings into resources |
| Message resolution | Pattern formatting with arguments |
| Error handling | Missing messages, resolution errors, fallback chains |
| Critical | `formatPattern()` requires the pattern from `getMessage().value`, NOT the message directly |

### 5. @fluent/react Integration
| Requirement | Detail |
|-------------|--------|
| LocalizationProvider | Context setup, bundle generation |
| Localized component | `<Localized>` for DOM overlays |
| useLocalization hook | Imperative message formatting |
| Bundle negotiation | `@fluent/langneg` for locale matching |
| Critical | React integration requires async bundle loading patterns |

### 6. Locale Management
| Requirement | Detail |
|-------------|--------|
| Language negotiation | `negotiateLanguages()` from @fluent/langneg |
| Fallback chains | Primary → fallback → default locale ordering |
| Dynamic switching | Runtime locale change without page reload |
| File organization | Directory structure for FTL files per locale |
| Critical | Locale codes MUST follow BCP 47 standard |

### 7. Advanced Patterns
| Requirement | Detail |
|-------------|--------|
| Message references | Messages referencing other messages |
| Parameterized terms | Terms with arguments for grammatical variants |
| Compound messages | Messages with value AND attributes |
| Bidirectional text | RTL/LTR isolation and handling |
| Critical | Message references vs term references have different behaviors |

---

## Critical Requirements (apply to ALL skills)

- All FTL code MUST conform to the Fluent specification at projectfluent.org
- All TypeScript MUST work with @fluent/bundle 0.18+, @fluent/react 0.15+
- FTL examples MUST use proper indentation and whitespace handling
- Integration skills MUST show both FTL files and TypeScript code (D-007)
- Code examples MUST be verified against official documentation

---

## Structural Requirements

### Skill Format
- SKILL.md < 500 lines (heavy content in references/)
- YAML frontmatter with name and description (including trigger words)
- English-only content
- Deterministic language (ALWAYS/NEVER, imperative)

### Skill Categories
| Category | Purpose | Must Include |
|----------|---------|--------------|
| syntax/ | How to write FTL | FTL patterns, select expressions, terms, version notes |
| impl/ | How to integrate | Bundle setup, React integration, locale management |
| errors/ | How to handle failures | Parse errors, resolution failures, debugging |
| core/ | Cross-cutting | API overview, architecture, concepts |
| agents/ | Orchestration | Validation checklists, auto-detection |

---

## Research Requirements (before creating any skill)

1. Official Fluent specification MUST be consulted and referenced
2. @fluent/bundle source code MUST be checked for accuracy when docs are ambiguous
3. Anti-patterns MUST be identified from real issues (GitHub issues, forums)
4. FTL examples MUST be verified against the specification (not hallucinated)
5. Version accuracy MUST be confirmed via WebFetch (D-009)

---

## Non-Requirements (explicitly out of scope)

- No ICU MessageFormat coverage (Fluent is an alternative, not a wrapper)
- No i18next, react-intl, or other i18n library comparisons
- No Pontoon (Mozilla's translation platform) administration guides
- No Fluent Rust crate coverage (fluent-rs) — this package covers the JavaScript ecosystem
- No server-side rendering deep-dives beyond basic patterns
- No coverage of deprecated @fluent/bundle versions before 0.18
