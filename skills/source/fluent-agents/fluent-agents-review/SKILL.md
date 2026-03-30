---
name: fluent-agents-review
description: >
  Use when reviewing or validating generated FTL and TypeScript Fluent code for correctness.
  Prevents shipping broken translations, wrong API usage, and common Fluent anti-patterns.
  Covers FTL syntax compliance, whitespace rules, select expressions, term visibility, and React integration.
  Keywords: code review, FTL validation, anti-patterns, select expression,
  term visibility, API audit, check translations, review FTL file,
  validate i18n code, translation quality.
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0, @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-agents-review

## Quick Reference

### Review Scope

| Area | What to Check | Severity |
|------|--------------|----------|
| FTL Syntax | Whitespace, identifiers, comments, special chars | Critical |
| Select Expressions | Default variant, CLDR categories, no ICU syntax | Critical |
| Terms | `-` prefix, no API access, literal-only args | Critical |
| Attributes | Value-less messages handled, `attrs` declared in React | High |
| API Usage | Two-step flow, null checks, errors array | Critical |
| React Integration | Provider wrapping, Localized props, hook usage | High |
| Error Handling | addResource errors logged, formatPattern errors collected | High |
| Anti-Patterns | All 16+ consolidated patterns checked | Medium-Critical |

### Critical Warnings

**NEVER** approve FTL that uses tab characters for indentation. Tabs are treated as text content, not whitespace. ALWAYS require spaces (U+0020) only.

**NEVER** approve FTL that omits the default variant (`*[other]`) in a select expression. The grammar mandates exactly one default variant.

**NEVER** approve TypeScript that calls `bundle.formatPattern()` without an errors array in production code. Omitting it causes the app to throw on the first resolution error.

**NEVER** approve code that accesses terms via `getMessage("-brand-name")`. Terms are private and ONLY resolved through message references.

**NEVER** approve FTL that uses ICU MessageFormat syntax (`{count, plural, one{...}}`). Fluent uses `{ $count -> [one] ... *[other] ... }`.

---

## FTL Syntax Validation Checklist

### Whitespace and Indentation

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Indentation characters | Spaces (U+0020) only | Tabs used — treated as text, breaks layout |
| Continuation lines | Indented by at least 1 space | Unindented line ends the pattern |
| First char after indent | NOT `[`, `*`, or `.` | Parsed as variant key, default marker, or attribute |
| Escape for special start chars | Use `{"["}`, `{"*"}`, `{"."}` | Raw special char causes parse error |
| Literal curly braces | Use `{"{"}` and `{"}"}` | Raw `{` or `}` breaks placeable parsing |

### Identifiers

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Message ID format | Starts with `[a-zA-Z]`, then `[a-zA-Z0-9_-]*` | Starting with number, using dots or spaces |
| Term ID format | `-` prefix + same rules as message ID | Missing `-` prefix on terms |
| Attribute ID format | `.` prefix + identifier | Space between `.` and name |

### Comments

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Space after `#` | `# comment` (space required) | `#comment` — invalid syntax |
| Comment levels | `#` (message), `##` (group), `###` (resource) | Wrong level for context |
| Binding rule | `#` comment directly above message binds to it | Blank line between comment and message breaks binding |

### Select Expressions

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Default variant | Exactly one `*[key]` per select | Missing default — syntax error |
| CLDR categories | `zero`, `one`, `two`, `few`, `many`, `other` | Invented categories like `none`, `single`, `multiple` |
| Selector syntax | `{ $var -> [key] ... }` | ICU syntax `{var, plural, ...}` |
| Arrow operator | `->` (hyphen + greater-than) | `=>` or other arrow styles |
| Variant indentation | Each variant on a new indented line | Variants on same line |

### Terms

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Term prefix | `-term-name` (hyphen prefix) | Missing hyphen — becomes a message |
| Term MUST have value | Pattern is required (unlike messages) | Attribute-only term — invalid |
| Named arg values | String literals or number literals only | Variable reference as arg: `-term(case: $var)` — invalid |
| Term attributes | Private — usable ONLY as selectors | Attempting to display term attribute directly |

### Built-in Functions

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Function names | `NUMBER()`, `DATETIME()` (uppercase convention) | `number()`, `Plural()` — wrong names |
| NUMBER options | `minimumFractionDigits`, `maximumFractionDigits`, `type`, etc. | ICU options like `style: "currency"` |
| DATETIME options | `day`, `month`, `year`, `hour`, `minute`, etc. | Non-Intl options |
| Ordinal usage | `NUMBER($pos, type: "ordinal")` | Plain `$pos` in selector for ordinals |

---

## API Usage Validation Checklist

### Bundle Setup

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Resource creation | `new FluentResource(ftlString)` | Passing raw string to `addResource()` |
| addResource return | Check returned `Error[]` array | Silently ignoring parse errors |
| allowOverrides | Explicit `{ allowOverrides: true }` when intended | Duplicate IDs produce errors without flag |
| Bundle locale | Valid BCP 47 string(s) | Empty string or invalid locale |

### Message Retrieval and Formatting

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Two-step flow | `getMessage()` then `formatPattern()` | Calling nonexistent `bundle.format()` (removed in v0.14.0) |
| Null check on getMessage | `if (msg?.value)` before formatting | `msg.value!` — crashes on attribute-only messages |
| Errors array in production | `formatPattern(pattern, args, errors)` | Omitting errors array — throws on first error |
| Variable types | `string`, `number`, `Date`, `FluentType`, `TemporalObject` | Passing objects, arrays, or booleans |
| Term access | Terms resolve automatically in messages | `getMessage("-brand")` — returns undefined |

### Error Handling

| Check | Expected | Common Failure |
|-------|----------|----------------|
| addResource errors | Log to console or monitoring | Silently discarding |
| formatPattern errors | Collect in array, log in production | No errors array — app crashes |
| Missing message fallback | Check `hasMessage()` or handle undefined | Calling `formatPattern` on undefined |
| Cyclic reference | Errors array catches `RangeError` | Unhandled crash |

---

## React Integration Validation Checklist

### Provider Setup

| Check | Expected | Common Failure |
|-------|----------|----------------|
| Provider wrapping | `<LocalizationProvider l10n={l10n}>` wraps app | Localized components outside provider |
| ReactLocalization instance | Created via `new ReactLocalization(bundles)` | Passing raw bundles array to provider |
| Bundle generation | Stable reference (useState/useMemo) | Creating new ReactLocalization on every render |
| Async loading | All bundles ready before rendering provider | Rendering before translations load |

### Localized Component

| Check | Expected | Common Failure |
|-------|----------|----------------|
| `id` prop | Message identifier string | Missing `id` — no translation lookup |
| `attrs` prop | `{{ title: true, placeholder: true }}` | Missing attrs — attributes silently not translated |
| `vars` prop | `{{ userName: "Anna" }}` | Wrong variable names vs. FTL `$variables` |
| `elems` prop | Maps to FTL `<tag>text</tag>` markup | Element keys don't match FTL tag names |
| Fallback child | Meaningful English fallback element | Empty child or no child |

### Hook Usage

| Check | Expected | Common Failure |
|-------|----------|----------------|
| useLocalization | `const { l10n } = useLocalization()` | Destructuring wrong property |
| getString | `l10n.getString(id, vars, fallback)` | Using nonexistent `l10n.format()` |
| Imperative use | For alerts, confirms, logging | Using hook output in JSX instead of `<Localized>` |
| HOC avoidance | Prefer `useLocalization` hook | Using `withLocalization` — no ref forwarding |

---

## Common AI Mistake Detection

When reviewing AI-generated Fluent code, ALWAYS check for these patterns:

| AI Mistake | What It Looks Like | Correct Fluent |
|------------|-------------------|----------------|
| ICU plural syntax | `{count, plural, one{...} other{...}}` | `{ $count -> [one] ... *[other] ... }` |
| Wrong variable syntax | `{name}` or `{{name}}` or `%{name}` | `{ $name }` |
| Missing `$` on variables | `{ name }` (message reference) | `{ $name }` (variable reference) |
| No default variant | `{ $count -> [one] One [other] More }` | `{ $count -> [one] One *[other] More }` |
| Tab indentation | Tab characters in FTL | Spaces only |
| Direct term access | `bundle.getMessage("-brand")` | Reference in FTL: `{ -brand }` |
| Old API usage | `bundle.format()`, `bundle.addMessages()` | `getMessage()` + `formatPattern()`, `addResource()` |
| String to addResource | `bundle.addResource("hello = Hi")` | `bundle.addResource(new FluentResource("hello = Hi"))` |

---

## Review Execution Order

When performing a Fluent code review, ALWAYS follow this sequence:

1. **Scan FTL files** — whitespace, identifiers, comments, special characters
2. **Validate select expressions** — default variant, CLDR categories, syntax
3. **Check terms** — prefix, visibility, argument constraints
4. **Check attributes** — value-less messages, React attrs mapping
5. **Review TypeScript** — two-step API, null checks, error handling
6. **Review React code** — provider setup, Localized props, hook usage
7. **Run anti-pattern scan** — check all items in [references/anti-patterns.md](references/anti-patterns.md)
8. **Check for AI mistakes** — ICU confusion, wrong variable syntax, missing defaults

Report findings grouped by severity: Critical > High > Medium > Low.

---

## Decision Trees

### Is This FTL Valid?

```
Message starts with letter? ─── NO ──→ INVALID (must start [a-zA-Z])
  │ YES
  ▼
Uses spaces for indent? ─── NO ──→ INVALID (tabs are text, not indent)
  │ YES
  ▼
Select has *[default]? ─── NO ──→ INVALID (default variant mandatory)
  │ YES
  ▼
Term has value (not attr-only)? ─── NO ──→ INVALID (terms MUST have value)
  │ YES
  ▼
Named args are literals? ─── NO ──→ INVALID (no variable refs in named args)
  │ YES
  ▼
VALID FTL
```

### Is This TypeScript Integration Correct?

```
Uses new FluentResource()? ─── NO ──→ FIX (raw strings not accepted)
  │ YES
  ▼
Checks addResource errors? ─── NO ──→ FIX (log parse errors)
  │ YES
  ▼
Uses getMessage + formatPattern? ─── NO ──→ FIX (no bundle.format())
  │ YES
  ▼
Null-checks msg.value? ─── NO ──→ FIX (value can be null)
  │ YES
  ▼
Passes errors array? ─── NO ──→ FIX (throws without it)
  │ YES
  ▼
CORRECT INTEGRATION
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete validation checklist organized by area
- [references/examples.md](references/examples.md) -- Review scenarios with good code, bad code, and fixes
- [references/anti-patterns.md](references/anti-patterns.md) -- Consolidated list of ALL anti-patterns (16+)

### Official Sources

- https://projectfluent.org/fluent/guide/
- https://github.com/projectfluent/fluent.js
- https://github.com/projectfluent/fluent/blob/master/spec/fluent.ebnf
