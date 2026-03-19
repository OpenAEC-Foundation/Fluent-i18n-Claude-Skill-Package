# Project Fluent Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized from raw masterplan after vooronderzoek review.
Date: 2026-03-19

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-01 | **Merged** `fluent-syntax-advanced` into `syntax-messages` and `syntax-functions` | "Advanced" is too vague. Message references and complex placeables belong in `syntax-messages`. Bidirectional text and Unicode isolation belong in `syntax-functions` (controlled via NUMBER/DATETIME and useIsolating). |
| D-02 | **Merged** `fluent-impl-dom-overlays` into `fluent-impl-react` | DOM overlays are the `elems` and `attrs` props on `<Localized>` — they are part of the React component API, not a separate domain. The research shows they share the same source module. |
| D-03 | **Merged** `fluent-errors-debugging` into `fluent-errors-resolution` | Debugging workflow is cross-cutting and thin. Console warnings, fallback behavior, and missing translation detection are resolution-time concerns. Combining produces a more complete error skill. |

**Result**: 19 raw skills → **16 definitive skills** (3 merges, 0 additions, 0 removals).

---

## Definitive Skill Inventory (16 skills)

### fluent-core/ (3 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `fluent-core-architecture` | Design philosophy; asymmetric localization; FTL format overview; bundle/resource/message model; Fluent vs ICU MessageFormat comparison; key terminology | FluentBundle, FluentResource, Message, Pattern | Vooronderzoek §1.1, §1.11 | M | None |
| `fluent-core-bundle-api` | FluentBundle constructor+options; addResource; getMessage; hasMessage; formatPattern; FluentResource; error handling modes; FluentType hierarchy; useIsolating; transform | FluentBundle, FluentResource, formatPattern, getMessage, FluentNumber, FluentDateTime, FluentNone | Vooronderzoek §2.2-§2.5, §2.7 | L | None |
| `fluent-core-langneg` | negotiateLanguages(); 3 strategies (filtering/matching/lookup); BCP 47; acceptedLanguages(); fallback chains; defaultLocale | negotiateLanguages, acceptedLanguages, filterMatches | Vooronderzoek §3.7 | S | None |

### fluent-syntax/ (5 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `fluent-syntax-messages` | Simple/multiline messages; placeables; variable references `{ $var }`; message references; text elements; whitespace/indentation/dedentation rules; comments (# ## ###); special characters; EBNF grammar excerpts | Message, Pattern, Identifier, VariableReference, MessageReference, CommentLine | Vooronderzoek §1.2-§1.3, §1.7, §1.10, §1.12 | M | core-architecture |
| `fluent-syntax-selectors` | Select expressions; CLDR plural categories (zero/one/two/few/many/other); default variant `*[other]`; ordinal plurals; exact numeric matches; formatted selectors; string selectors; nested selectors | SelectExpression, Variant, DefaultVariant, VariantKey, NUMBER(type:"ordinal") | Vooronderzoek §1.4 | M | syntax-messages |
| `fluent-syntax-terms` | Terms `-term-name`; term references; parameterized terms (named args only); term attributes for grammatical metadata; visibility rules (terms NEVER in API); term vs message comparison table | Term, TermReference, CallArguments, NamedArgument | Vooronderzoek §1.5 | M | syntax-messages |
| `fluent-syntax-attributes` | Message attributes `.attr`; term attributes; value-less messages (attribute-only); compound messages (value + attributes); accessing attributes via getMessage().attributes; attribute access in FTL | Attribute, AttributeAccessor, msg.attributes | Vooronderzoek §1.6 | S | syntax-messages |
| `fluent-syntax-functions` | Built-in NUMBER() with all Intl.NumberFormat options; DATETIME() with all Intl.DateTimeFormat options; custom function registration; FluentFunction signature; function call grammar; bidirectional text handling via useIsolating | NUMBER, DATETIME, FluentFunction, FunctionReference, CallArguments | Vooronderzoek §1.8-§1.9, §2.5 | M | syntax-messages, core-bundle-api |

### fluent-impl/ (4 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `fluent-impl-react` | LocalizationProvider setup; Localized component (id, vars, elems, attrs props); DOM overlays with elems; attribute mapping with attrs; useLocalization hook; ReactLocalization class; withLocalization HOC (deprecated); parseMarkup for SSR; bundle generation patterns (array + generator) | LocalizationProvider, Localized, useLocalization, ReactLocalization, MarkupParser | Vooronderzoek §3.2-§3.6, §3.9 | L | core-bundle-api |
| `fluent-impl-locale-loading` | Async FTL file loading via fetch; lazy bundle generation with generators; CachedSyncIterable; caching strategies; multiple FTL files per locale; file organization per locale; code-splitting patterns | fetch, FluentResource, FluentBundle, addResource | Vooronderzoek §3.2.3, §3.11 | M | core-bundle-api |
| `fluent-impl-locale-switching` | Dynamic locale change without reload; React state pattern; negotiateLanguages integration; new ReactLocalization on switch; user preference storage (localStorage); server-side locale detection via acceptedLanguages | useState, negotiateLanguages, ReactLocalization, acceptedLanguages | Vooronderzoek §3.6, §3.7, §3.9 | M | impl-react, core-langneg |
| `fluent-impl-project-setup` | Complete project scaffolding; npm dependencies; directory structure; minimal working FTL+TS+React setup; semantic message ID conventions; file naming; tooling (@fluent/syntax for validation) | npm install, FluentBundle, FluentResource, LocalizationProvider | Vooronderzoek §3.8, §3.11, §2.6 | M | core-bundle-api, impl-react |

### fluent-errors/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `fluent-errors-parsing` | FTL parse errors; indentation errors (tabs vs spaces); missing default variant; special char restrictions ([*. at line start); junk handling; comment space requirement; identifier restrictions; @fluent/syntax FluentParser for strict validation; addResource error array | FluentResource, FluentParser, Junk, Annotation, addResource errors | Vooronderzoek §1.10, §1.12, §2.3, §2.6 | M | syntax-messages |
| `fluent-errors-resolution` | Runtime resolution errors (10 error types); missing messages/variables/terms; type mismatches; cyclic references; excessive placeables; error collection mode vs throw mode; fallback behavior ({???}); console warnings; debugging workflow; missing translation detection | formatPattern errors, ReferenceError, TypeError, RangeError, FluentNone | Vooronderzoek §2.4, §2.10 | M | core-bundle-api |

### fluent-agents/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `fluent-agents-review` | Validation checklist for FTL + TypeScript; anti-pattern detection (all 16+ anti-patterns); spec compliance checks; common AI mistakes (ICU confusion, wrong variable syntax); whitespace validation; dual-coverage verification | All validation rules | Vooronderzoek §1.12, §2.10, §3.10 | M | ALL syntax + impl skills |
| `fluent-agents-project-scaffolder` | Generate complete Fluent project: FTL files per locale, bundle setup, React integration with LocalizationProvider, locale negotiation, async loading, file organization | All scaffolding patterns | Vooronderzoek §3.8, §3.11 | L | ALL core + impl skills |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-architecture`, `core-bundle-api`, `core-langneg` | 3 | None | Foundation, no deps |
| 2 | `syntax-messages`, `syntax-selectors`, `syntax-terms` | 3 | Batch 1 | Core FTL syntax |
| 3 | `syntax-attributes`, `syntax-functions`, `impl-react` | 3 | Batch 1-2 | Remaining syntax + React integration |
| 4 | `impl-locale-loading`, `impl-locale-switching`, `impl-project-setup` | 3 | Batch 1, 3 | Implementation skills |
| 5 | `errors-parsing`, `errors-resolution` | 2 | Batch 1-2 | Error handling |
| 6 | `agents-review`, `agents-project-scaffolder` | 2 | ALL above | Agent skills last |

**Total**: 16 skills across 6 batches.

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package
RESEARCH_FILE = C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\docs\research\vooronderzoek-fluent.md
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md
```

---

### Batch 1

#### Prompt: fluent-core-architecture

```
## Task: Create the fluent-core-architecture skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-core\fluent-core-architecture\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (key types: FluentBundle, FluentResource, Message, Pattern)
3. references/examples.md (basic FTL + TS examples showing the full flow)
4. references/anti-patterns.md (ICU confusion, wrong mental model)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: fluent-core-architecture
description: "Guides Project Fluent architecture and design philosophy including asymmetric localization, FTL format overview, the bundle-resource-message model, and how Fluent differs from ICU MessageFormat. Activates when working with Project Fluent, understanding FTL files, or reasoning about localization architecture."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Fluent design philosophy: asymmetric localization, progressive enhancement
- Core concept: translators can add complexity independently of developers
- FTL format overview: messages, terms, attributes, selectors, comments
- The runtime model: FluentBundle → FluentResource → Message → Pattern → formatPattern
- Key terminology: message, term, pattern, placeable, variant, attribute
- Fluent vs ICU MessageFormat comparison table (11 key differences)
- Key packages overview: @fluent/bundle, @fluent/react, @fluent/langneg, @fluent/syntax

### Research Sections to Read
From vooronderzoek-fluent.md:
- §1.1: Design Philosophy
- §1.11: Key Differences from ICU MessageFormat
- §2.1: Package Overview
- §3.1: @fluent/react Overview

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must be verified against vooronderzoek research
- Include version annotations (@fluent/bundle 0.19.1, Fluent Syntax 1.0)
- Include Critical Warnings section: NEVER confuse Fluent with ICU MessageFormat
```

#### Prompt: fluent-core-bundle-api

```
## Task: Create the fluent-core-bundle-api skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-core\fluent-core-bundle-api\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete FluentBundle API: constructor, addResource, getMessage, hasMessage, formatPattern; FluentResource; FluentType hierarchy)
3. references/examples.md (bundle creation, resource loading, message formatting, error collection, custom functions)
4. references/anti-patterns.md (all 10 API anti-patterns from research)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: fluent-core-bundle-api
description: "Guides @fluent/bundle API including FluentBundle constructor with options, addResource for loading FTL, getMessage and hasMessage for retrieval, formatPattern for rendering translations, FluentResource parsing, error handling modes, custom functions, and the FluentType hierarchy. Activates when using FluentBundle, formatting messages, handling translation errors, or registering custom Fluent functions."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- FluentBundle constructor: locales (string|string[]), options (functions, useIsolating, transform)
- addResource(resource, {allowOverrides}): returns Error[], per-message error recovery
- getMessage(id): returns {value: Pattern|null, attributes: Record<string,Pattern>} | undefined
- hasMessage(id): boolean check
- formatPattern(pattern, args, errors): the core formatting method
- FluentResource: constructor(source), body property, optimized runtime parser
- Error handling: two modes (throw vs collect), 10 error types table
- Custom functions: FluentFunction signature, registration, NUMBER/DATETIME builtins with allowed options
- FluentType hierarchy: FluentNone, FluentNumber, FluentDateTime
- FluentVariable type: string | number | Date | FluentType | TemporalObject

### Research Sections to Read
From vooronderzoek-fluent.md:
- §2.2: FluentBundle Class (all subsections)
- §2.3: FluentResource Class
- §2.4: Error Handling
- §2.5: Custom Functions
- §2.7: Type Exports and TypeScript Usage
- §2.10: Anti-Patterns (AP-01 through AP-10)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include complete method signature table
- Include error types diagnostic table
- MUST show the two-step getMessage+formatPattern pattern prominently
- Include Critical Warnings: NEVER pass raw string to addResource, ALWAYS check msg.value for null
```

#### Prompt: fluent-core-langneg

```
## Task: Create the fluent-core-langneg skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-core\fluent-core-langneg\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (negotiateLanguages full signature, acceptedLanguages, filterMatches)
3. references/examples.md (client-side negotiation, server-side with Accept-Language, all 3 strategies)
4. references/anti-patterns.md (missing defaultLocale, wrong strategy choice)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: fluent-core-langneg
description: "Guides @fluent/langneg language negotiation including negotiateLanguages() with filtering/matching/lookup strategies, BCP 47 locale handling, acceptedLanguages() for server-side HTTP header parsing, and fallback chain construction. Activates when selecting user locales, configuring language fallback, parsing Accept-Language headers, or setting up locale negotiation."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/langneg 0.7+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- negotiateLanguages(requested, available, options): full signature with all parameters
- Three strategies: filtering (all matches), matching (best per request), lookup (single best)
- defaultLocale behavior: appended if not in results; REQUIRED for lookup strategy
- acceptedLanguages(header): parse HTTP Accept-Language header, q-value sorting
- filterMatches: low-level matching engine
- BCP 47 locale identifiers: language, language-region, language-script
- Subtag matching: "de-DE" matches "de" as fallback
- Decision tree: when to use which strategy

### Research Sections to Read
From vooronderzoek-fluent.md:
- §3.7: @fluent/langneg Overview (all subsections)
- §3.9: SSR Considerations (acceptedLanguages usage)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include strategy comparison table with example inputs/outputs
- Include decision tree for strategy selection
- Include Critical Warning: lookup strategy REQUIRES defaultLocale or throws
```

---

### Batch 2

#### Prompt: fluent-syntax-messages

```
## Task: Create the fluent-syntax-messages skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-syntax\fluent-syntax-messages\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (EBNF grammar excerpts: Message, Pattern, Identifier, VariableReference, MessageReference, CommentLine)
3. references/examples.md (simple messages, multiline, block text, placeables, variables, message references, comments, special characters)
4. references/anti-patterns.md (whitespace mistakes, tab usage, missing comment space, special char at line start)

### YAML Frontmatter
---
name: fluent-syntax-messages
description: "Guides FTL message syntax including simple and multiline messages, placeables with variable references, message references, text elements, whitespace and indentation rules, the dedentation algorithm, three-level comment system, and special character handling. Activates when writing FTL messages, formatting multiline text, using variables in translations, or understanding FTL whitespace rules."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Simple messages: key = Value pattern
- Multiline messages: continuation line indentation rules, block text format
- Dedentation algorithm: common indent stripped, first line excluded
- Placeables: { } syntax for dynamic content
- Variable references: { $variable-name } syntax, automatic locale-aware formatting
- Message references: { other-message } for embedding messages
- Comments: # (message), ## (group), ### (resource) — space required after #
- Whitespace rules: ONLY spaces for indent (NOT tabs), leading/trailing blank handling, interior blank preservation
- Special characters: [*. restricted at line start, { } must be escaped, escape sequences in quoted text only
- Identifier rules: starts with letter, contains [a-zA-Z0-9_-]
- EBNF grammar excerpts for key productions

### Research Sections to Read
From vooronderzoek-fluent.md:
- §1.2: Message Syntax
- §1.3: Placeables and Variables
- §1.7: Comments
- §1.10: Whitespace and Indentation Rules
- §1.12: Edge Cases and Gotchas (#1-#5, #8-#10)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- MUST include whitespace rules as Critical Warning (this is the #1 AI mistake)
- Include EBNF excerpts for Message, Pattern, Identifier
- Tab warning MUST be prominent: tabs are NOT whitespace in FTL
```

#### Prompt: fluent-syntax-selectors

```
## Task: Create the fluent-syntax-selectors skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-syntax\fluent-syntax-selectors\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (EBNF grammar: SelectExpression, Variant, DefaultVariant, VariantKey; CLDR category table per language)
3. references/examples.md (cardinal plurals, ordinal plurals, string selectors, exact numeric matches, formatted selectors, nested selectors)
4. references/anti-patterns.md (missing default variant, wrong CLDR categories, ICU plural syntax confusion)

### YAML Frontmatter
---
name: fluent-syntax-selectors
description: "Guides FTL select expressions including CLDR plural categories, default variant requirement, cardinal and ordinal plurals, exact numeric matches, string selectors, formatted number selectors, and nested selectors. Activates when writing plural forms, gender selectors, or any variant-based translation in FTL."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Select expression syntax: { $var -> [key] pattern *[default] pattern }
- CLDR plural categories: zero, one, two, few, many, other
- Default variant: MANDATORY, marked with *, grammar enforces exactly one
- Cardinal plurals: [one] [other] for English, more categories for other languages
- Ordinal plurals: NUMBER($var, type: "ordinal") for 1st/2nd/3rd/4th
- Exact numeric matches: [0] [1] checked BEFORE CLDR categories
- Formatted selectors: NUMBER($score, minimumFractionDigits: 1) -> [0.0] ...
- String selectors: direct string matching against variant keys
- Nested selectors: multi-level select expressions
- EBNF grammar: SelectExpression, variant_list, Variant, DefaultVariant, VariantKey

### Research Sections to Read
From vooronderzoek-fluent.md:
- §1.4: Select Expressions and Plurals (complete section)
- §1.12: Edge Case #7 (default variant mandatory)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Critical Warning: EVERY select expression MUST have exactly one *[default] variant
- Critical Warning: NEVER use ICU syntax {count, plural, ...} — Fluent uses { $count -> ... }
- Include CLDR category table for at least English, French, Arabic, Polish
```

#### Prompt: fluent-syntax-terms

```
## Task: Create the fluent-syntax-terms skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-syntax\fluent-syntax-terms\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (EBNF grammar: Term, TermReference, CallArguments, NamedArgument; term vs message comparison)
3. references/examples.md (basic terms, parameterized terms for grammar cases, term attributes for gender, term references in messages)
4. references/anti-patterns.md (accessing terms via API, variable refs in named args, treating terms as messages)

### YAML Frontmatter
---
name: fluent-syntax-terms
description: "Guides FTL terms including the -term-name syntax, term references in messages, parameterized terms with named arguments for grammatical cases, term attributes for gender and animacy metadata, and the critical visibility rule that terms are NEVER accessible via the runtime API. Activates when defining brand names, shared vocabulary, or grammatical metadata in FTL."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Term syntax: -term-name = Value (hyphen prefix)
- Term references in messages: { -brand-name }
- Parameterized terms: { -brand-name(case: "locative") } — named args ONLY, values MUST be literals
- Term attributes: .gender = feminine for grammatical metadata
- Term attributes as selectors: { -brand-name.gender -> [masculine] ... [feminine] ... *[other] ... }
- Visibility rule: terms are NEVER returned by getMessage() — they are internal-only
- Term vs Message comparison table
- EBNF grammar: Term, TermReference, NamedArgument restrictions
- Terms MUST have a value (unlike messages which can be attribute-only)

### Research Sections to Read
From vooronderzoek-fluent.md:
- §1.5: Terms and References (complete section)
- §1.12: Edge Cases #6 (named argument restrictions), #10 (term must have value)
- §2.10: Anti-Pattern AP-05 (accessing terms via getMessage)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Critical Warning: NEVER try to access terms via bundle.getMessage("-term") — returns undefined
- Critical Warning: Named argument values MUST be string or number literals, NEVER variable references
```

---

### Batch 3

#### Prompt: fluent-syntax-attributes

```
## Task: Create the fluent-syntax-attributes skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-syntax\fluent-syntax-attributes\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (EBNF grammar: Attribute, AttributeAccessor; getMessage().attributes API)
3. references/examples.md (message attributes, term attributes, value-less messages, compound messages, accessing attributes in TypeScript)
4. references/anti-patterns.md (forgetting to check attributes, assuming value exists)

### YAML Frontmatter
---
name: fluent-syntax-attributes
description: "Guides FTL attributes including the .attribute syntax on messages and terms, value-less messages with only attributes, compound messages with value and attributes, accessing attributes via getMessage().attributes in TypeScript, and the Localized attrs prop in React. Activates when working with HTML element attributes, form field labels, or multi-value messages in FTL."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0, @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Attribute syntax: .attribute-name = Value (indented, starts with dot)
- Message attributes: grouping related strings (placeholder, aria-label, title)
- Term attributes: private metadata (gender, animacy) — ONLY usable as selectors
- Value-less messages: messages with ONLY attributes and no value (msg.value is null)
- Compound messages: messages with BOTH value and attributes
- TypeScript access: getMessage(id).attributes["attr-name"] returns Pattern
- React access: <Localized attrs={{ title: true }}> — explicit opt-in per attribute
- EBNF grammar: Attribute, AttributeAccessor

### Research Sections to Read
From vooronderzoek-fluent.md:
- §1.6: Attributes (complete section)
- §2.2.3: getMessage() — attributes in return value
- §3.3.3: Attribute Mapping in React
- §2.10: Anti-Pattern AP-03 (not checking null value)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include both FTL and TypeScript examples (dual coverage per D-007)
- Critical Warning: ALWAYS check msg.value for null — value-less messages are valid FTL
```

#### Prompt: fluent-syntax-functions

```
## Task: Create the fluent-syntax-functions skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-syntax\fluent-syntax-functions\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (NUMBER options table, DATETIME options table, FluentFunction signature, FunctionReference grammar)
3. references/examples.md (NUMBER formatting, DATETIME formatting, custom function registration, bidirectional text examples)
4. references/anti-patterns.md (wrong function syntax, passing objects as args, disabling useIsolating carelessly)

### YAML Frontmatter
---
name: fluent-syntax-functions
description: "Guides FTL built-in functions NUMBER() and DATETIME() with all Intl formatting options, custom function registration via FluentBundle constructor, the FluentFunction type signature, function call grammar, and Unicode bidirectional text isolation. Activates when formatting numbers, dates, currencies in FTL, creating custom Fluent functions, or handling RTL/LTR text mixing."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0, @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- NUMBER() function: all Intl.NumberFormat options (minimumFractionDigits, maximumFractionDigits, useGrouping, minimumIntegerDigits, currencyDisplay, etc.)
- NUMBER() in selectors vs placeables: different use cases
- DATETIME() function: all Intl.DateTimeFormat options (dateStyle, timeStyle, day, month, year, hour, minute, second, weekday, era, etc.)
- Custom function registration: FluentBundle constructor functions option
- FluentFunction signature: (positional: FluentValue[], named: Record<string, FluentValue>) => FluentValue
- Function call grammar: FunctionReference, CallArguments, Argument, NamedArgument
- Bidirectional text: useIsolating option, U+2068/U+2069 isolation marks
- When to disable useIsolating (testing, unidirectional content)

### Research Sections to Read
From vooronderzoek-fluent.md:
- §1.8: Built-in Functions
- §1.9: Custom Functions
- §2.5: Custom Functions (registration and built-ins)
- §2.2.1: Constructor options (useIsolating)
- §2.10: Anti-Pattern AP-08 (unsupported types), AP-10 (disabling useIsolating)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include both FTL and TypeScript examples (dual coverage per D-007)
- Include complete NUMBER and DATETIME option tables
- Critical Warning: NEVER pass objects/arrays/booleans as FluentVariable — only string, number, Date
```

#### Prompt: fluent-impl-react

```
## Task: Create the fluent-impl-react skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-impl\fluent-impl-react\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (LocalizationProvider props, Localized props, useLocalization return, ReactLocalization constructor+methods, MarkupParser type)
3. references/examples.md (provider setup, Localized basic, vars, elems/overlays, attrs, useLocalization imperative, bundle generation patterns)
4. references/anti-patterns.md (creating ReactLocalization in render, missing attrs, withLocalization vs hook, post-translation manipulation)

### YAML Frontmatter
---
name: fluent-impl-react
description: "Guides @fluent/react integration including LocalizationProvider setup, Localized component with id/vars/elems/attrs props, DOM overlays for rich translated markup, useLocalization hook for imperative formatting, ReactLocalization class API, bundle generation patterns, custom parseMarkup for SSR, and withLocalization HOC deprecation. Activates when integrating Fluent with React, rendering translations in components, using DOM overlays, or setting up the localization provider."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/react 0.15+, @fluent/bundle 0.16+, React 16.8+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- LocalizationProvider: l10n prop, React Context distribution
- Localized component: id, vars, elems, attrs, children (fallback) props
- DOM overlays: elems prop mapping custom element names to React components, template-based parsing
- Attribute mapping: attrs prop for explicit opt-in, security rationale
- useLocalization hook: returns { l10n: ReactLocalization }, getString(id, vars, fallback)
- ReactLocalization class: constructor(bundles, parseMarkup?, reportError?), getBundle, getString, getElement, areBundlesEmpty
- Bundle generation: array-based (eager) vs generator-based (lazy), CachedSyncIterable
- parseMarkup for SSR: custom parser using jsdom/cheerio when <template> unavailable
- withLocalization HOC: exists but DOES NOT support ref forwarding — prefer hooks
- Async loading pattern: fetch → FluentResource → FluentBundle → ReactLocalization

### Research Sections to Read
From vooronderzoek-fluent.md:
- §3.2: LocalizationProvider (all subsections)
- §3.3: Localized Component (all subsections)
- §3.4: useLocalization Hook
- §3.5: ReactLocalization Class
- §3.9: SSR Considerations
- §3.10: Anti-Patterns (all React-specific)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- MUST show both FTL and TypeScript/React examples (dual coverage per D-007)
- Include complete props table for Localized
- Include ReactLocalization method signatures
- Critical Warning: ALWAYS declare attrs explicitly — translations cannot inject arbitrary attributes
- Critical Warning: NEVER manipulate translated strings after formatting
```

---

### Batch 4

#### Prompt: fluent-impl-locale-loading

```
## Task: Create the fluent-impl-locale-loading skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-impl\fluent-impl-locale-loading\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (fetch pattern, FluentResource from string, addResource, multiple resources per bundle)
3. references/examples.md (async loading, lazy generator, multiple FTL files, caching with Map)
4. references/anti-patterns.md (creating FluentResource per message, not logging addResource errors, blocking render on all locales)

### YAML Frontmatter
---
name: fluent-impl-locale-loading
description: "Guides FTL file loading patterns including async fetch-based loading, lazy bundle generation with generators, CachedSyncIterable for caching, multiple FTL files per locale, file organization conventions, and code-splitting strategies. Activates when loading FTL files from a server, organizing translation files, implementing lazy locale loading, or optimizing translation bundle size."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Async loading: fetch() → response.text() → new FluentResource(text)
- Bundle creation: new FluentBundle(locale) → addResource(resource) → check errors
- Lazy generation: generator function yielding bundles on demand
- CachedSyncIterable: wraps generator for single-iteration caching
- Multiple FTL files per locale: repeated addResource() calls on same bundle
- File organization: /locales/{locale}/messages.ftl pattern
- Feature-based splitting: separate FTL files per page/feature
- Flat vs deep structure: when to use which
- Caching strategies: Map<string, FluentBundle>, pre-loading vs on-demand
- addResource error handling: log but don't throw

### Research Sections to Read
From vooronderzoek-fluent.md:
- §3.2.2: Bundle Generation Pattern
- §3.2.3: Async Bundle Loading
- §3.11: File Organization Best Practices
- §2.10: Anti-Pattern AP-06 (FluentResource per message), AP-09 (ignoring addResource errors)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include both FTL file structure and TypeScript loading code
- Include decision tree: flat files vs feature-split files
```

#### Prompt: fluent-impl-locale-switching

```
## Task: Create the fluent-impl-locale-switching skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-impl\fluent-impl-locale-switching\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (React state pattern, negotiateLanguages integration, localStorage API)
3. references/examples.md (complete locale switcher component, server-side detection, user preference persistence)
4. references/anti-patterns.md (recreating ReactLocalization on every render, hardcoding locales)

### YAML Frontmatter
---
name: fluent-impl-locale-switching
description: "Guides dynamic locale switching in Fluent React apps including the React state pattern for re-localization, negotiateLanguages integration for fallback chains, user preference persistence in localStorage, server-side locale detection with acceptedLanguages, and re-rendering strategies. Activates when implementing a language switcher, persisting user locale preferences, or handling locale changes without page reload."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/react 0.15+, @fluent/langneg 0.7+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- React state pattern: useState for currentLocales, new ReactLocalization on change
- negotiateLanguages integration: [userChoice] as requested, AVAILABLE_LOCALES as available
- Bundle regeneration: create new bundles for negotiated locales, pass to ReactLocalization
- Automatic re-rendering: React context propagates to all <Localized> components
- User preference persistence: localStorage for client, cookie for server
- Server-side detection: acceptedLanguages(req.headers["accept-language"])
- navigator.languages for initial detection
- Available locales constant: single source of truth

### Research Sections to Read
From vooronderzoek-fluent.md:
- §3.6: Locale Switching Patterns
- §3.7: @fluent/langneg (negotiation in context)
- §3.9: SSR Considerations (server-side locale detection)
- §3.10: Anti-Pattern #4 (creating ReactLocalization in render)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include complete locale switcher component example
- Include both client-side and server-side detection patterns
```

#### Prompt: fluent-impl-project-setup

```
## Task: Create the fluent-impl-project-setup skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-impl\fluent-impl-project-setup\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (npm packages, versions, peer dependencies)
3. references/examples.md (minimal working setup, complete project scaffold, directory structure)
4. references/anti-patterns.md (wrong package versions, missing peer deps, wrong file structure)

### YAML Frontmatter
---
name: fluent-impl-project-setup
description: "Guides complete Fluent project setup including npm dependency installation, directory structure for FTL files and TypeScript code, minimal working FTL+TypeScript+React configuration, semantic message ID conventions, @fluent/syntax for FTL validation tooling, and file naming best practices. Activates when starting a new Fluent project, adding Fluent to an existing app, or scaffolding the localization infrastructure."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+, @fluent/react 0.15+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- npm install: @fluent/bundle, @fluent/react, @fluent/langneg (required); @fluent/syntax (optional tooling)
- Version compatibility matrix: @fluent/bundle 0.18+, @fluent/react 0.15+, React 16.8+
- Directory structure: src/l10n/, public/locales/{locale}/, component organization
- Minimal working setup: 1 FTL file + bundle creation + LocalizationProvider + Localized
- Semantic message ID conventions: component-element-action pattern
- FTL file naming: feature-based (main.ftl, errors.ftl, settings.ftl)
- @fluent/syntax for tooling: FluentParser for validation, FluentSerializer for generation
- .ftl file extension convention

### Research Sections to Read
From vooronderzoek-fluent.md:
- §3.8: Complete Integration Pattern
- §3.11: File Organization Best Practices
- §2.6: @fluent/syntax Package (when to use for tooling)
- §3.1: Installation

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include step-by-step setup workflow
- Include complete directory structure diagram
- Include minimal working example (FTL + TS + React)
```

---

### Batch 5

#### Prompt: fluent-errors-parsing

```
## Task: Create the fluent-errors-parsing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-errors\fluent-errors-parsing\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (addResource error return, FluentParser API, Junk/Annotation types)
3. references/examples.md (common parse errors with FTL showing wrong/correct, @fluent/syntax validation)
4. references/anti-patterns.md (ignoring parse errors, using @fluent/syntax for runtime)

### YAML Frontmatter
---
name: fluent-errors-parsing
description: "Diagnoses and resolves FTL parse errors including indentation mistakes with tabs vs spaces, missing default variant in selectors, special character restrictions at line start, comment space requirement, identifier format violations, junk entry recovery, and @fluent/syntax FluentParser for strict validation. Activates when encountering FTL syntax errors, debugging broken translations, or validating FTL files."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0, @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- addResource() error array: per-message syntax errors, non-blocking
- Indentation errors: tabs treated as text not whitespace, wrong indent level
- Missing default variant: *[other] mandatory in every select expression
- Special characters at line start: [, *, . cannot start continuation lines
- Comment format: space required after # (# comment not #comment)
- Identifier restrictions: must start with letter, only [a-zA-Z0-9_-]
- Curly brace escaping: literal { } must use quoted text placeables
- Junk handling: parser produces Junk entries, does not fail entirely
- @fluent/syntax FluentParser: strict validation with Span positions and Annotation errors
- FluentSerializer: round-trip testing for FTL validation
- Escape sequences: only in quoted text, not in regular text

### Research Sections to Read
From vooronderzoek-fluent.md:
- §1.10: Whitespace and Indentation Rules
- §1.12: Edge Cases and Gotchas (all 10)
- §2.3: FluentResource (parser design)
- §2.6: @fluent/syntax Package
- §2.10: Anti-Pattern AP-07 (using @fluent/syntax for runtime), AP-09 (ignoring addResource errors)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Format as diagnostic table: Error → Cause → Fix
- Include FTL examples showing wrong and correct versions
- Include tab vs space comparison prominently
```

#### Prompt: fluent-errors-resolution

```
## Task: Create the fluent-errors-resolution skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-errors\fluent-errors-resolution\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (all 10 error types table, formatPattern error modes, FluentNone behavior)
3. references/examples.md (each error type with code showing cause and fix, debugging workflow)
4. references/anti-patterns.md (omitting errors array in production, not checking getMessage return, passing wrong types)

### YAML Frontmatter
---
name: fluent-errors-resolution
description: "Diagnoses and resolves Fluent runtime resolution errors including missing messages, missing variables, missing terms, type mismatches, cyclic references, excessive placeables, the two error handling modes (throw vs collect), fallback behavior with {???} placeholders, and debugging workflow for missing translations. Activates when encountering runtime translation errors, debugging missing or broken translations, or implementing error handling for Fluent formatting."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- 10 resolution error types: ReferenceError (variable, message, term, attribute, no-value, function), TypeError (unsupported type, non-callable), RangeError (cyclic, no-default, excessive placeables)
- Two error modes: silent collection (errors array) vs throw on first (no array)
- Fallback behavior: {$name}, {msg-id}, {-term-id}, {???} placeholders
- Excessive placeables (>100): FATAL, terminates resolution, always throws
- FluentNone: the "missing value" type, displays as {???}
- Missing message debugging: hasMessage check → getMessage null check → value null check
- Missing variable: passes through as {$varName} in output
- Console warnings: via ReactLocalization reportError callback
- Development vs production error handling strategy
- Parse-time vs resolution-time error distinction

### Research Sections to Read
From vooronderzoek-fluent.md:
- §2.4: Error Handling (complete section, all 10 error types)
- §2.10: Anti-Patterns AP-03 (null value), AP-04 (omitting errors array), AP-05 (terms via getMessage), AP-08 (wrong types)
- §3.5: ReactLocalization reportError

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Format as diagnostic table: Error Type → Condition → Fallback → Fix
- Include both TypeScript error handling patterns
- Critical Warning: ALWAYS provide errors array in production to prevent crashes
```

---

### Batch 6

#### Prompt: fluent-agents-review

```
## Task: Create the fluent-agents-review skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-agents\fluent-agents-review\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete validation checklist organized by area)
3. references/examples.md (review scenarios: good code, bad code, fixes)
4. references/anti-patterns.md (consolidated list of ALL anti-patterns from all skills)

### YAML Frontmatter
---
name: fluent-agents-review
description: "Validates generated FTL and TypeScript code for Project Fluent correctness by checking FTL syntax compliance, whitespace rules, select expression structure, term visibility, API usage patterns, error handling, React integration, and known anti-patterns. Activates when reviewing Fluent code, validating FTL files, checking TypeScript integration code, or auditing a Fluent project for correctness."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0, @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- FTL syntax validation: whitespace (spaces only), indentation, identifier format, comment syntax
- Select expression checks: default variant present, valid CLDR categories, no ICU syntax
- Term checks: -prefix present, not accessed via API, named args are literals only
- Attribute checks: value-less messages handled, attrs explicitly declared in React
- API usage: two-step getMessage+formatPattern, null checks, errors array in production
- React integration: LocalizationProvider wrapping, Localized props, useLocalization hook
- Error handling: addResource errors logged, formatPattern errors collected
- Anti-pattern detection: all 16+ anti-patterns from vooronderzoek consolidated
- Common AI mistakes: ICU syntax confusion, wrong variable syntax, missing FTL indentation

### Research Sections to Read
From vooronderzoek-fluent.md:
- §1.12: Edge Cases and Gotchas (all 10)
- §2.10: Anti-Patterns (all 10)
- §3.10: Common Issues and Anti-Patterns (all 6)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Structure as a runnable checklist: Area → Check Item → Expected → Common Failure
- Group checks: FTL Syntax, API Usage, React Integration, Error Handling
```

#### Prompt: fluent-agents-project-scaffolder

```
## Task: Create the fluent-agents-project-scaffolder skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Fluent-i18n-Claude-Skill-Package\skills\source\fluent-agents\fluent-agents-project-scaffolder\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (file templates, dependency list, configuration reference)
3. references/examples.md (complete scaffolded project output with all files)
4. references/anti-patterns.md (scaffolding mistakes)

### YAML Frontmatter
---
name: fluent-agents-project-scaffolder
description: "Generates a complete Project Fluent localization setup including FTL files per locale, FluentBundle configuration, React integration with LocalizationProvider, locale negotiation with @fluent/langneg, async loading infrastructure, locale switching mechanism, and proper directory structure. Activates when scaffolding a new Fluent project, adding localization to an existing React app, or generating the complete i18n infrastructure."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+, @fluent/react 0.15+, @fluent/langneg 0.7+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Complete file listing: package.json deps, l10n.ts, App.tsx wrapper, FTL files per locale
- npm dependencies: exact packages and peer dependency requirements
- Directory structure: src/l10n/, public/locales/{locale}/, component integration
- FTL file templates: main.ftl with common messages, semantic IDs
- Bundle setup: FluentBundle creation, FluentResource loading, error handling
- React integration: LocalizationProvider, async init pattern
- Locale negotiation: negotiateLanguages with navigator.languages
- Locale switching: state-based pattern with preference persistence
- Decision tree: which packages to include based on project type (React, vanilla JS, SSR)

### Research Sections to Read
From vooronderzoek-fluent.md:
- §3.8: Complete Integration Pattern
- §3.11: File Organization Best Practices
- §3.6: Locale Switching Patterns
- §3.7: negotiateLanguages

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include complete file listing with content templates
- Generated code MUST follow all patterns from syntax and impl skills
- Include decision tree: React app vs vanilla JS vs Node.js server
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── fluent-core/
│   ├── fluent-core-architecture/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── methods.md
│   │       ├── examples.md
│   │       └── anti-patterns.md
│   ├── fluent-core-bundle-api/
│   │   ├── SKILL.md
│   │   └── references/
│   └── fluent-core-langneg/
│       ├── SKILL.md
│       └── references/
├── fluent-syntax/
│   ├── fluent-syntax-messages/
│   ├── fluent-syntax-selectors/
│   ├── fluent-syntax-terms/
│   ├── fluent-syntax-attributes/
│   └── fluent-syntax-functions/
├── fluent-impl/
│   ├── fluent-impl-react/
│   ├── fluent-impl-locale-loading/
│   ├── fluent-impl-locale-switching/
│   └── fluent-impl-project-setup/
├── fluent-errors/
│   ├── fluent-errors-parsing/
│   └── fluent-errors-resolution/
└── fluent-agents/
    ├── fluent-agents-review/
    └── fluent-agents-project-scaffolder/
```
