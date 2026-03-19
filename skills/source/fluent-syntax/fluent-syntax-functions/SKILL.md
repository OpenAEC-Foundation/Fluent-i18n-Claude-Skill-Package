---
name: fluent-syntax-functions
description: "Guides FTL built-in functions NUMBER() and DATETIME() with all Intl formatting options, custom function registration via FluentBundle constructor, the FluentFunction type signature, function call grammar, and Unicode bidirectional text isolation. Activates when formatting numbers, dates, currencies in FTL, creating custom Fluent functions, or handling RTL/LTR text mixing."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0, @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-syntax-functions

## Quick Reference

### Built-in Functions

| Function | Purpose | Backed By | Usage Context |
|----------|---------|-----------|---------------|
| `NUMBER()` | Format numeric values | `Intl.NumberFormat` | Placeables and selectors |
| `DATETIME()` | Format date/time values | `Intl.DateTimeFormat` | Placeables only |

### Function Call Grammar (EBNF)

```ebnf
FunctionReference ::= Identifier CallArguments
CallArguments     ::= blank? "(" blank? argument_list blank? ")"
argument_list     ::= (Argument blank? "," blank?)* Argument?
Argument          ::= NamedArgument | InlineExpression
NamedArgument     ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
```

Function names are plain `Identifier` values -- ALWAYS use uppercase by convention: `NUMBER`, `DATETIME`, `MY_FUNC`.

### FluentFunction Type Signature

```typescript
type FluentFunction = (
  positional: Array<FluentValue>,
  named: Record<string, FluentValue>
) => FluentValue;
```

Where `FluentValue = FluentType<unknown> | string`.

### Critical Warnings

**NEVER** pass objects, arrays, or booleans as `FluentVariable` values -- only `string`, `number`, `Date`, `FluentType`, or `TemporalObject` are valid. Passing unsupported types causes a `TypeError` at resolution time.

**NEVER** disable `useIsolating` for bundles that handle RTL/LTR mixed content -- Unicode isolation marks (U+2068/U+2069) prevent bidirectional text corruption.

**NEVER** use lowercase for function names in FTL -- the parser treats lowercase identifiers as message references, not function calls. ALWAYS use UPPERCASE.

**NEVER** pass variable references as named argument values in FTL -- named arguments MUST be string literals or number literals: `NUMBER($x, minimumFractionDigits: 2)` is correct, `NUMBER($x, minimumFractionDigits: $digits)` is NOT valid.

---

## NUMBER() Function

### In Placeables (Display Formatting)

```ftl
# Basic usage -- explicit formatting
emails = You have { NUMBER($count) } unread emails.

# With formatting options
price = Total: { NUMBER($amount, minimumFractionDigits: 2, maximumFractionDigits: 2) }
file-size = Size: { NUMBER($bytes, useGrouping: 0) } bytes
balance = Balance: { NUMBER($value, minimumIntegerDigits: 2) }
```

### In Selectors (Plural Category Determination)

```ftl
# Cardinal plurals (default)
emails =
    { $count ->
        [one] You have one unread email.
       *[other] You have { $count } unread emails.
    }

# Ordinal plurals
your-rank = { NUMBER($pos, type: "ordinal") ->
    [1] You finished first!
    [one] You finished {$pos}st
    [two] You finished {$pos}nd
    [few] You finished {$pos}rd
   *[other] You finished {$pos}th
}

# Formatted number as selector
your-score =
    { NUMBER($score, minimumFractionDigits: 1) ->
        [0.0] You scored zero points.
       *[other] You scored { NUMBER($score, minimumFractionDigits: 1) } points.
    }
```

Exact numeric matches (like `[1]`) are checked BEFORE CLDR plural category matches. ALWAYS place exact matches above category matches.

### NUMBER() Options (Complete)

See [references/methods.md](references/methods.md) for the full options table.

---

## DATETIME() Function

### Basic Usage

```ftl
# Short date
today = Today is { DATETIME($date, month: "long", day: "numeric", year: "numeric") }.

# Time display
current-time = Time: { DATETIME($now, hour: "numeric", minute: "2-digit") }.

# Full date-time with style shortcuts
last-login = Last login: { DATETIME($date, dateStyle: "long", timeStyle: "short") }.

# Weekday display
meeting-day = Meeting on { DATETIME($date, weekday: "long") }.
```

### TypeScript -- Passing Date Variables

```typescript
const msg = bundle.getMessage("today");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { date: new Date() });
  // "Today is March 19, 2026."
}
```

ALWAYS pass `Date` objects (or `number` timestamps or Temporal objects) as date variables. NEVER pass pre-formatted date strings -- let Fluent handle locale-aware formatting.

### DATETIME() Options (Complete)

See [references/methods.md](references/methods.md) for the full options table.

---

## Custom Functions

### Registration via FluentBundle Constructor

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US", {
  functions: {
    CURRENCY: (positional, named) => {
      const value = positional[0];
      const num = typeof value === "number"
        ? value
        : Number(value);
      const currency = named["currency"];
      const currencyCode = typeof currency === "string"
        ? currency
        : "USD";
      return new Intl.NumberFormat("en-US", {
        style: "currency",
        currency: currencyCode,
      }).format(num);
    },
    UPCASE: (positional, named) => {
      const val = positional[0];
      return typeof val === "string"
        ? val.toUpperCase()
        : String(val).toUpperCase();
    },
    GREETING_TIME: (positional, named) => {
      const hour = new Date().getHours();
      if (hour < 12) return "morning";
      if (hour < 18) return "afternoon";
      return "evening";
    },
  },
});
```

Corresponding FTL:

```ftl
total = Total: { CURRENCY($amount, currency: "EUR") }
shout = { UPCASE("hello world") }
greeting = Good { GREETING_TIME() }!
```

### Custom Function Rules

1. **ALWAYS** register functions in the `functions` option of the `FluentBundle` constructor
2. Custom functions are merged with built-in `NUMBER` and `DATETIME` -- naming a custom function `NUMBER` or `DATETIME` overrides the built-in
3. Functions receive `positional` (array of `FluentValue`) and `named` (record of `FluentValue`) arguments
4. Functions MUST return a `FluentValue` (`string` or `FluentType<unknown>`)
5. If a function referenced in FTL is not registered, resolution produces a `ReferenceError` and the fallback `{FUNC_NAME()}` is displayed

### Decision Tree -- When to Use Custom Functions

```
Need locale-aware number formatting?
├── YES → Use NUMBER() with Intl.NumberFormat options
│   ├── Need currency? → NUMBER($val, style: "currency") or custom CURRENCY()
│   ├── Need percentage? → Custom PERCENT() function
│   └── Need ordinals? → NUMBER($val, type: "ordinal") in selector
│
Need locale-aware date formatting?
├── YES → Use DATETIME() with Intl.DateTimeFormat options
│
Need text transformation?
├── YES → Custom function (UPCASE, LOWERCASE, CAPITALIZE)
│
Need runtime logic (time of day, platform detection)?
├── YES → Custom function returning a string for selector matching
│
Need formatting not covered by Intl?
├── YES → Custom function with your own formatting logic
```

---

## Bidirectional Text Isolation

### How useIsolating Works

When `useIsolating` is `true` (the default), every placeable value is wrapped in Unicode isolation marks:

- **U+2068** (First Strong Isolate) -- inserted BEFORE the placeable
- **U+2069** (Pop Directional Isolate) -- inserted AFTER the placeable

This prevents RTL text in placeables from corrupting the surrounding LTR text layout (and vice versa).

### Example -- With Isolation (Default)

```typescript
const bundle = new FluentBundle("en-US"); // useIsolating: true (default)
const resource = new FluentResource(`welcome = Welcome, { $name }!`);
bundle.addResource(resource);

const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "Ahmed" });
  // Internal: "Welcome, \u2068Ahmed\u2069!"
  // Renders correctly in both LTR and RTL contexts
}
```

### When to Disable useIsolating

ONLY disable `useIsolating` in these cases:

1. **Unit testing** -- Unicode marks interfere with string equality assertions
2. **Unidirectional content** -- ALL content is the same direction (e.g., English-only app)
3. **Pre-isolated content** -- Placeables already contain isolation marks

```typescript
// For testing
const bundle = new FluentBundle("en-US", { useIsolating: false });

// For production with mixed directionality -- KEEP DEFAULT
const bundle = new FluentBundle("ar", { useIsolating: true });
```

**NEVER** disable `useIsolating` for Arabic, Hebrew, or any locale that mixes RTL and LTR scripts. Doing so causes placeable values to reorder surrounding text unpredictably.

---

## Reference Links

- [references/methods.md](references/methods.md) -- NUMBER options table, DATETIME options table, FluentFunction signature, FunctionReference grammar
- [references/examples.md](references/examples.md) -- NUMBER formatting, DATETIME formatting, custom function registration, bidirectional text
- [references/anti-patterns.md](references/anti-patterns.md) -- Wrong function syntax, passing objects as args, disabling useIsolating carelessly

### Official Sources

- https://projectfluent.org/fluent/guide/builtins.html
- https://projectfluent.org/fluent/guide/functions.html
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/builtins.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/bundle.ts
- https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf
