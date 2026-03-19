---
name: fluent-core-bundle-api
description: "Guides @fluent/bundle API including FluentBundle constructor with options, addResource for loading FTL, getMessage and hasMessage for retrieval, formatPattern for rendering translations, FluentResource parsing, error handling modes, custom functions, and the FluentType hierarchy. Activates when using FluentBundle, formatting messages, handling translation errors, or registering custom Fluent functions."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-core-bundle-api

## Quick Reference

### FluentBundle Method Signatures

| Method | Signature | Returns |
|--------|-----------|---------|
| `constructor` | `(locales: string \| string[], options?)` | `FluentBundle` |
| `addResource` | `(res: FluentResource, {allowOverrides?})` | `Error[]` |
| `getMessage` | `(id: string)` | `Message \| undefined` |
| `hasMessage` | `(id: string)` | `boolean` |
| `formatPattern` | `(pattern, args?, errors?)` | `string` |

### Constructor Options

| Option | Type | Default | Purpose |
|--------|------|---------|---------|
| `functions` | `Record<string, FluentFunction>` | `{}` | Custom functions available in FTL |
| `useIsolating` | `boolean` | `true` | Wrap placeables in Unicode bidi isolation marks |
| `transform` | `TextTransform` | `(v) => v` | Transform applied to all text elements |

### FluentType Hierarchy

| Class | Extends | Value Type | Purpose |
|-------|---------|------------|---------|
| `FluentType<T>` | -- | `T` | Abstract base for all Fluent types |
| `FluentNone` | `FluentType<string>` | `string` | Represents missing/failed values, renders as `{???}` |
| `FluentNumber` | `FluentType<number>` | `number` | Locale-aware number formatting via `Intl.NumberFormat` |
| `FluentDateTime` | `FluentType<number>` | `number` | Locale-aware date/time formatting via `Intl.DateTimeFormat` |

### FluentVariable Accepted Types

| Type | Example |
|------|---------|
| `string` | `{ name: "Anna" }` |
| `number` | `{ count: 5 }` |
| `Date` | `{ date: new Date() }` |
| `FluentType` | `{ amount: new FluentNumber(42.5, { minimumFractionDigits: 2 }) }` |
| `TemporalObject` | `{ instant: Temporal.Now.instant() }` (v0.19.0+) |

### Critical Warnings

**NEVER** pass a raw string to `addResource()` -- it expects a `FluentResource` instance. ALWAYS wrap FTL strings: `bundle.addResource(new FluentResource(ftlSource))`.

**ALWAYS** use the two-step pattern: call `getMessage()` first, then `formatPattern()` on its value. There is no single-call `format()` method (removed in v0.14.0).

**ALWAYS** check `msg.value` for `null` before formatting -- messages with only attributes have `value: null`. Formatting `null` causes a runtime error.

**ALWAYS** pass an `errors` array to `formatPattern()` in production code. Without it, the method **throws** on the first resolution error instead of returning a best-effort string.

**NEVER** access terms via `getMessage("-term-name")` -- this returns `undefined`. Terms are resolved automatically when referenced inside messages.

**NEVER** pass objects, arrays, or booleans as variables -- only `string`, `number`, `Date`, `FluentType`, and `TemporalObject` are valid `FluentVariable` types.

---

## Core Pattern: Two-Step Message Formatting

Every message formatting operation in Fluent follows this mandatory two-step pattern:

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

// Step 0: Create bundle and load resource
const bundle = new FluentBundle("en-US");
const errors = bundle.addResource(new FluentResource(`
welcome = Welcome, {$name}!
login-input =
    .placeholder = Email address
    .aria-label = Login input
`));

// Step 1: Retrieve the message
const msg = bundle.getMessage("welcome");

// Step 2: Format the pattern (with error collection)
if (msg?.value) {
  const errs: Error[] = [];
  const text = bundle.formatPattern(msg.value, { name: "Anna" }, errs);
  // text === "Welcome, Anna!"
}
```

For attribute-only messages, access attributes directly:

```typescript
const loginMsg = bundle.getMessage("login-input");
if (loginMsg) {
  // loginMsg.value is null -- this message has no value
  const placeholder = bundle.formatPattern(loginMsg.attributes["placeholder"]);
  // placeholder === "Email address"
}
```

---

## Error Handling Modes

### Mode 1: Silent Collection (Production)

```typescript
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
// Returns best-effort string with {???} for failed parts
// errors array contains all resolution issues
```

### Mode 2: Throw on First Error (Development)

```typescript
// No errors array -- throws ReferenceError, TypeError, or RangeError
const text = bundle.formatPattern(pattern, args);
```

### Error Types Diagnostic Table

| Error Type | Trigger | Fallback Output |
|-----------|---------|-----------------|
| `ReferenceError` | Unknown variable `$name` | `{$name}` |
| `ReferenceError` | Unknown message reference | `{msg-id}` |
| `ReferenceError` | Unknown term reference | `{-term-id}` |
| `ReferenceError` | Unknown attribute | `{msg.attr}` |
| `ReferenceError` | No value on message (attributes-only) | `{msg-id}` |
| `ReferenceError` | Unknown function | `{FUNC()}` |
| `TypeError` | Unsupported variable type | `{$name}` |
| `TypeError` | Function is not callable | `{FUNC()}` |
| `RangeError` | Cyclic reference detected | `{???}` |
| `RangeError` | No default variant in select | `{???}` |
| `RangeError` | Excessive placeables (>100) | **Fatal -- throws even with errors array** |

### Parse-Time vs. Resolution-Time Errors

- **Parse-time** (`addResource()` return value): FTL syntax errors. Per-message -- one bad message does not block others.
- **Resolution-time** (`formatPattern()` errors): Missing variables, broken references, type mismatches. Occur when formatting a specific pattern.

---

## Decision Tree: Choosing Error Handling Mode

```
Is this production code?
├── YES → ALWAYS pass errors array to formatPattern()
│         Log errors for monitoring, display best-effort text to users
└── NO (development/testing)
    ├── Want to catch errors early? → Omit errors array (throws)
    └── Want to see fallback behavior? → Pass errors array
```

## Decision Tree: FluentResource vs. @fluent/syntax

```
What do you need?
├── Runtime message formatting → @fluent/bundle (FluentResource)
│   Optimized parser, small bundle, fast
├── FTL linting or validation → @fluent/syntax (FluentParser)
│   Full AST with spans, strict parsing
├── Programmatic FTL generation → @fluent/syntax (FluentSerializer)
│   Round-trip parse/serialize
└── Editor tooling → @fluent/syntax
    Span information for highlights, completions
```

## Decision Tree: Bundle Constructor Options

```
Setting up FluentBundle options?
├── useIsolating
│   ├── Mixed RTL/LTR content (e.g., Arabic UI with English names)
│   │   → ALWAYS keep true (default)
│   ├── Single-direction content only
│   │   → Safe to set false
│   └── Testing (Unicode marks interfere with assertions)
│       → Set false for test bundles only
├── functions
│   ├── Need custom formatting beyond NUMBER/DATETIME?
│   │   → Register in constructor: { functions: { MY_FUNC: ... } }
│   └── Built-in NUMBER and DATETIME sufficient?
│       → No configuration needed
└── transform
    ├── Need to sanitize or escape all text elements?
    │   → Provide transform function
    └── No text transformation needed?
        → Use default identity transform
```

---

## Custom Functions

ALWAYS register custom functions in the constructor. Functions receive two arguments: positional values and named options.

```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    UPCASE: (positional: FluentValue[], named: Record<string, FluentValue>) => {
      const val = positional[0];
      return typeof val === "string" ? val.toUpperCase() : String(val).toUpperCase();
    },
  },
});
```

Corresponding FTL:
```ftl
shout = { UPCASE($name) }
```

### Built-in Functions

| Function | Delegates To | Purpose |
|----------|-------------|---------|
| `NUMBER()` | `Intl.NumberFormat` | Locale-aware number formatting |
| `DATETIME()` | `Intl.DateTimeFormat` | Locale-aware date/time formatting |

FTL usage:
```ftl
price = Total: { NUMBER($amount, minimumFractionDigits: 2) }
event-date = { DATETIME($date, dateStyle: "long") }
```

---

## FluentResource

`FluentResource` parses an FTL string into an optimized runtime representation. It uses a purpose-built parser that is smaller and faster than `@fluent/syntax`.

```typescript
const resource = new FluentResource(`
hello = Hello, world!
welcome = Welcome, {$name}!
-brand = Acme Corp
`);
// resource.body.length === 3 (2 messages + 1 term)
```

Key characteristics:
- Parsing errors are per-message -- one bad message does not prevent others from being parsed
- ALWAYS use the constructor directly. `FluentResource.fromString()` was removed in v0.14.0
- For strict FTL validation, use `@fluent/syntax` instead

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete API signatures for FluentBundle, FluentResource, FluentType hierarchy
- [references/examples.md](references/examples.md) -- Working code examples for bundle creation, formatting, error handling, custom functions
- [references/anti-patterns.md](references/anti-patterns.md) -- All 10 API anti-patterns with corrections

### Official Sources

- https://github.com/projectfluent/fluent.js/tree/main/fluent-bundle
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/bundle.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/resolver.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/builtins.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/CHANGELOG.md
