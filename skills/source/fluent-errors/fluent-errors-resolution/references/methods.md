# Resolution Error Methods Reference

## Complete Error Type Inventory

### ReferenceError Variants (6 types)

| # | Variant | FTL Trigger | Fallback | Error Message Pattern |
|---|---------|------------|----------|----------------------|
| 1 | Unknown variable | `{$undeclaredVar}` | `{$undeclaredVar}` | `Unknown variable: $undeclaredVar` |
| 2 | Unknown message | `{other-msg}` (not defined) | `{other-msg}` | `Unknown message: other-msg` |
| 3 | Unknown term | `{-missing-term}` (not defined) | `{-missing-term}` | `Unknown term: -missing-term` |
| 4 | Unknown attribute | `{msg.missing-attr}` | `{msg.missing-attr}` | `Unknown attribute: msg.missing-attr` |
| 5 | No value | `{msg-id}` (msg has only attributes) | `{msg-id}` | `No value: msg-id` |
| 6 | Unknown function | `{MISSING()}` | `{MISSING()}` | `Unknown function: MISSING()` |

### TypeError Variants (2 types)

| # | Variant | TypeScript Trigger | Fallback | Error Message Pattern |
|---|---------|-------------------|----------|----------------------|
| 7 | Unsupported type | `args: { user: { name: "x" } }` | `{$user}` | Variable type not supported |
| 8 | Not callable | `functions: { FOO: "not a function" }` | `{FOO()}` | Function is not callable |

### RangeError Variants (3 types)

| # | Variant | FTL Trigger | Fallback | Fatal? |
|---|---------|------------|----------|--------|
| 9 | Cyclic reference | `a = {b}` + `b = {a}` | `{???}` | No |
| 10 | No default variant | `{$x -> [one] ... }` (missing `*`) | `{???}` | No |
| 11 | Excessive placeables | >100 placeables in one pattern | **Throws** | **YES** |

---

## formatPattern Error Modes

### Signature

```typescript
formatPattern(
  pattern: Pattern,
  args?: Record<string, FluentVariable> | null,
  errors?: Array<Error> | null
): string
```

### Mode Comparison

| Aspect | Silent Collection (`errors` provided) | Throw Mode (`errors` omitted) |
|--------|--------------------------------------|-------------------------------|
| Behavior on error | Appends to `errors` array, continues | Throws immediately |
| Return value | Best-effort string with fallback placeholders | N/A (exception thrown) |
| Multiple errors | All collected | Only first error surfaces |
| Excessive placeables | **Still throws** (exception to collection mode) | Throws |
| Use case | Production | Development, testing |

### Error Array Lifecycle

```typescript
// The errors array is APPENDED TO, not replaced
const errors: Error[] = [];

// First call might add 2 errors
bundle.formatPattern(pattern1, args1, errors);
// errors.length could be 2

// Second call adds to the SAME array
bundle.formatPattern(pattern2, args2, errors);
// errors.length could now be 5

// ALWAYS create a fresh array per formatting call if you want per-message errors
const errorsPerMessage: Error[] = [];
bundle.formatPattern(pattern, args, errorsPerMessage);
```

---

## FluentNone Behavior

### Class Definition

```typescript
class FluentNone extends FluentType<string> {
  constructor(value?: string); // default value: "???"
  toString(scope: Scope): string; // returns `{${this.value}}`
}
```

### When FluentNone Is Created

| Situation | FluentNone.value |
|-----------|-----------------|
| Unknown variable `$name` | `$name` |
| Unknown message `msg-id` | `msg-id` |
| Unknown term `-term-id` | `-term-id` |
| Unknown attribute `msg.attr` | `msg.attr` |
| No value on message | `msg-id` |
| Unknown function `FUNC()` | `FUNC()` |
| Cyclic reference | `???` |
| No default variant | `???` |

### FluentNone Propagation

FluentNone propagates through the resolution pipeline:

```
FTL: price = Total: { NUMBER($amount, minimumFractionDigits: 2) }

If $amount is missing:
1. Resolver creates FluentNone("$amount")
2. NUMBER() receives FluentNone as input
3. NUMBER() returns FluentNone (does not crash)
4. Final output: "Total: {$amount}"
```

This propagation ensures that a single missing variable does not corrupt the entire message -- only the affected placeable shows a fallback.

---

## addResource Error Return

```typescript
addResource(
  res: FluentResource,
  options?: { allowOverrides?: boolean }
): Array<Error>
```

Parse-time errors are returned as an `Error[]`:
- Each error corresponds to one malformed message in the FTL source
- Valid messages in the same resource are still added successfully
- These are SYNTAX errors, not resolution errors

### Parse Error vs Resolution Error

| Property | Parse Error (addResource) | Resolution Error (formatPattern) |
|----------|--------------------------|----------------------------------|
| When | At resource loading time | At formatting time |
| Cause | Invalid FTL syntax | Missing refs, wrong types, cycles |
| Scope | Per-message in resource | Per-pattern being formatted |
| Recovery | Other messages still load | Other parts of pattern still resolve |
| Collection | Always returned as array | Only collected if errors array provided |

---

## getMessage and hasMessage for Error Diagnosis

### getMessage Return Shape

```typescript
interface Message {
  value: Pattern | null;      // null if message has only attributes
  attributes: Record<string, Pattern>;
}
```

### Diagnostic Checks

```typescript
// Check 1: Message exists?
bundle.hasMessage(id): boolean
// false -> message not in any loaded resource

// Check 2: Retrieve message
bundle.getMessage(id): Message | undefined
// undefined -> same as hasMessage returning false

// Check 3: Value exists?
msg.value: Pattern | null
// null -> message has only attributes (e.g., .placeholder, .aria-label)

// Check 4: Attribute exists?
msg.attributes["name"]: Pattern | undefined
// undefined -> attribute not defined on this message
```

### Terms Are Not Accessible

```typescript
// Terms (prefixed with -) are NEVER returned by getMessage or hasMessage
bundle.hasMessage("-brand-name");  // ALWAYS returns false
bundle.getMessage("-brand-name");  // ALWAYS returns undefined

// Terms are only resolved internally when referenced in message patterns
// There is NO public API to format a term directly
```

---

## ReactLocalization.reportError

```typescript
constructor(
  bundles: Iterable<FluentBundle>,
  parseMarkup?: MarkupParser | null,
  reportError?: (error: Error) => void  // default: console.warn
)
```

The `reportError` callback is invoked when:
- `getString()` encounters resolution errors during formatting
- `getElement()` encounters resolution errors during formatting
- A message ID is not found in any bundle

This is the React-layer error surface. It wraps the bundle-level `formatPattern` errors and provides a single callback for monitoring integration.

```typescript
const l10n = new ReactLocalization(
  bundles,
  null,
  (error) => {
    // Log to external monitoring
    Sentry.captureException(error);
    // Also log locally for debugging
    console.warn("[Fluent]", error.message);
  }
);
```
