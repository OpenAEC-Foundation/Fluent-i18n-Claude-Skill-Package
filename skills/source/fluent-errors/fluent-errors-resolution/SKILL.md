---
name: fluent-errors-resolution
description: >
  Use when FluentBundle.formatPattern returns unexpected results, fallback values, or {???} placeholders.
  Prevents undiagnosed runtime errors from missing messages, variables, terms, and cyclic references.
  Covers throw vs collect error modes, type mismatches, excessive placeables, and fallback debugging.
  Keywords: formatPattern, resolution error, missing message, cyclic reference,
  fallback, error handling, translation shows ???, missing translation,
  wrong output, variable not replaced.
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-errors-resolution

## Quick Reference

### Resolution Error Diagnostic Table

| Error Type | Condition | Fallback Output | Fix |
|-----------|-----------|----------------|-----|
| `ReferenceError` | Unknown variable `$name` | `{$name}` | Pass the variable in `args`: `{ name: "value" }` |
| `ReferenceError` | Unknown message `msg-id` | `{msg-id}` | Add the message to the FTL resource or fix the ID typo |
| `ReferenceError` | Unknown term `-term-id` | `{-term-id}` | Add the term to the FTL resource or fix the ID typo |
| `ReferenceError` | Unknown attribute `msg.attr` | `{msg.attr}` | Add the attribute to the message definition |
| `ReferenceError` | No value on message (only attributes) | `{msg-id}` | Add a value pattern or use `.attribute` reference |
| `ReferenceError` | Unknown function `FUNC()` | `{FUNC()}` | Register the function in `FluentBundle` options |
| `TypeError` | Unsupported variable type (object, array, boolean) | `{$name}` | Pass only `string`, `number`, `Date`, or `FluentType` |
| `TypeError` | Function is not callable | `{FUNC()}` | Ensure the registered function matches `FluentFunction` signature |
| `RangeError` | Cyclic reference detected | `{???}` | Break the circular dependency between messages/terms |
| `RangeError` | No default variant in select expression | `{???}` | Add `*[other]` default variant to select expression |
| `RangeError` | Excessive placeables (>100) -- **FATAL** | **Throws** | Reduce placeable count; refactor into multiple messages |

### Two Error Handling Modes

**Mode 1 -- Silent collection (ALWAYS use in production):**
```typescript
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
// text = best-effort output with fallback placeholders for failed parts
// errors = array of ReferenceError, TypeError, or RangeError instances
if (errors.length > 0) {
  errors.forEach(e => console.warn("Resolution error:", e.message));
}
```

**Mode 2 -- Throw on first error (development/testing only):**
```typescript
// No errors array provided -> throws immediately on first resolution error
const text = bundle.formatPattern(pattern, args);
// Throws ReferenceError, TypeError, or RangeError
```

### Critical Warnings

**ALWAYS** provide the `errors` array parameter in production code. Without it, `formatPattern` **throws** on the first resolution error, crashing the application.

**NEVER** ignore the errors array after formatting. ALWAYS log or report errors so missing translations are caught early.

**ALWAYS** distinguish parse-time errors (from `addResource`) from resolution-time errors (from `formatPattern`). They are separate error surfaces with different handling patterns.

**NEVER** assume `getMessage()` returns a value pattern. Messages with only attributes have `value: null`. ALWAYS check `msg?.value` before calling `formatPattern`.

---

## Parse-Time vs Resolution-Time Errors

| Phase | Method | Error Source | Behavior |
|-------|--------|-------------|----------|
| Parse-time | `addResource()` | FTL syntax errors | Returns `Error[]`; per-message (one bad message does not break others) |
| Resolution-time | `formatPattern()` | Missing refs, wrong types, cycles | Collects in `errors` array OR throws (depending on mode) |

Parse errors and resolution errors are completely independent. A message can parse successfully but fail at resolution time due to missing variables or broken references.

---

## FluentNone: The Missing Value Type

When a reference fails during resolution, the resolver creates a `FluentNone` instance:

```typescript
class FluentNone extends FluentType<string> {
  constructor(value?: string); // default: "???"
  toString(scope: Scope): string; // returns `{${this.value}}`
}
```

- `FluentNone` is the "null object" of Fluent resolution
- Built-in functions (`NUMBER`, `DATETIME`) propagate `FluentNone` gracefully -- if the input is `FluentNone`, they return `FluentNone`
- The string representation is always `{value}` where value is the identifier that failed (e.g., `{$name}`, `{msg-id}`, `{???}`)

---

## Missing Message Debugging Workflow

ALWAYS follow this three-step check when a translation is missing:

```typescript
// Step 1: Does the bundle know about this message?
if (!bundle.hasMessage("welcome-user")) {
  // Message ID not in any loaded resource
  // Fix: check FTL file is loaded, check for ID typos
  return;
}

// Step 2: Can we retrieve the message object?
const msg = bundle.getMessage("welcome-user");
if (!msg) {
  // Should not happen if hasMessage returned true
  return;
}

// Step 3: Does the message have a value pattern?
if (!msg.value) {
  // Message has only attributes (e.g., .placeholder, .aria-label)
  // Fix: add a value pattern OR use msg.attributes["attrName"]
  return;
}

// Step 4: Format with error collection
const errors: Error[] = [];
const text = bundle.formatPattern(msg.value, { name: "Anna" }, errors);
if (errors.length > 0) {
  // Resolution errors: missing variables, broken refs, type issues
  errors.forEach(e => console.error("Resolution:", e.message));
}
```

---

## Decision Tree: Diagnosing a Broken Translation

```
Translation shows unexpected output
├── Output is the message ID itself (e.g., "welcome-user")
│   └── Message not found in any bundle
│       ├── FTL file not loaded? -> Check addResource() was called
│       ├── ID typo? -> Compare FTL file with code reference
│       └── Wrong bundle? -> Check locale negotiation result
├── Output contains {$variableName}
│   └── Variable not provided in args
│       ├── Typo in variable name? -> Compare FTL $name with args key
│       └── Missing from args object? -> Add the variable
├── Output contains {msg-id} or {-term-id}
│   └── Referenced message or term not defined
│       ├── Missing from FTL file? -> Add the message/term
│       └── In a different resource? -> Ensure both resources loaded in same bundle
├── Output contains {???}
│   └── Cyclic reference OR no default variant
│       ├── Cyclic: message A references B which references A -> Break the cycle
│       └── No default: select expression missing *[other] -> Add default variant
├── Output contains {FUNC()}
│   └── Custom function not registered
│       └── Register in FluentBundle constructor options.functions
└── formatPattern throws (no errors array)
    ├── RangeError: excessive placeables (>100) -> FATAL, refactor message
    └── Any other error -> Add errors array parameter
```

---

## Development vs Production Error Strategy

### Development

```typescript
// Throw on errors to catch issues early
try {
  const text = bundle.formatPattern(msg.value!, args);
} catch (e) {
  console.error("Fluent resolution error:", e);
  throw e; // Fail fast in development
}
```

### Production

```typescript
// Collect errors, never crash
const errors: Error[] = [];
const text = bundle.formatPattern(msg.value!, args, errors);
if (errors.length > 0) {
  // Report to monitoring (Sentry, DataDog, etc.)
  errors.forEach(e => reportError(e));
}
// text always returns a usable string, even with fallback placeholders
```

### React: ReactLocalization reportError

```typescript
import { ReactLocalization } from "@fluent/react";

const l10n = new ReactLocalization(
  bundles,
  null, // parseMarkup
  (error: Error) => {
    // Called when getString/getElement encounters resolution errors
    // Default: console.warn
    monitoringService.captureError(error);
  }
);
```

---

## Excessive Placeables (>100): The Fatal Error

This is the ONLY resolution error that is **always fatal** regardless of error mode. When a pattern contains more than 100 placeables, the resolver terminates immediately and throws a `RangeError`.

This limit exists to prevent denial-of-service from maliciously crafted FTL content. It cannot be configured or disabled.

**Fix:** Refactor the message into multiple smaller messages and compose them in application code, not in FTL.

---

## Reference Links

- [references/methods.md](references/methods.md) -- All 10 error types table, formatPattern error modes, FluentNone behavior
- [references/examples.md](references/examples.md) -- Each error type with cause and fix code, debugging workflow
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes in error handling

### Official Sources

- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/resolver.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/bundle.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-react/src/localization.ts
