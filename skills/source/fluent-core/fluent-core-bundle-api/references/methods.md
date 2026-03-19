# FluentBundle API — Complete Method Reference

## FluentBundle Class

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/bundle.ts

### Constructor

```typescript
constructor(
  locales: string | Array<string>,
  options?: {
    functions?: Record<string, FluentFunction>;
    useIsolating?: boolean;
    transform?: TextTransform;
  }
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `locales` | `string \| Array<string>` | (required) | BCP 47 locale identifier(s) used by `Intl` formatters |
| `functions` | `Record<string, FluentFunction>` | `{}` | Custom functions available in FTL, merged with built-in NUMBER and DATETIME |
| `useIsolating` | `boolean` | `true` | Wrap placeables in Unicode isolation marks (U+2068/U+2069) for bidirectional text |
| `transform` | `TextTransform` | `(v) => v` | Transform applied to all TextElement strings in patterns |

### addResource()

```typescript
addResource(
  res: FluentResource,
  options?: { allowOverrides?: boolean }
): Array<Error>
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `res` | `FluentResource` | (required) | A parsed FTL resource |
| `allowOverrides` | `boolean` | `false` | If `true`, new messages/terms overwrite existing ones. If `false`, duplicates produce an error. |

**Returns**: Array of `Error` objects for messages with syntax errors. Errors are per-message -- they do NOT prevent other valid messages from being added.

### getMessage()

```typescript
getMessage(id: string): Message | undefined
```

Returns the parsed message for the given identifier, or `undefined` if not found. Terms (identifiers starting with `-`) are NOT accessible via this method.

**Message shape**:
```typescript
interface Message {
  value: Pattern | null;
  attributes: Record<string, Pattern>;
}
```

- `value` -- the main pattern, or `null` if the message only has attributes
- `attributes` -- attribute patterns keyed by attribute name

### hasMessage()

```typescript
hasMessage(id: string): boolean
```

Returns `true` if the bundle contains a message with the given identifier. Does NOT check for terms.

### formatPattern()

```typescript
formatPattern(
  pattern: Pattern,
  args?: Record<string, FluentVariable> | null,
  errors?: Array<Error> | null
): string
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `pattern` | `Pattern` | (required) | A pattern from `getMessage().value` or `getMessage().attributes[name]` |
| `args` | `Record<string, FluentVariable> \| null` | `null` | Variables to substitute into the pattern |
| `errors` | `Array<Error> \| null` | `null` | If provided, resolution errors are appended instead of thrown |

**Error behavior**: If `errors` is NOT provided, throws on the first resolution error. If `errors` IS provided, collects errors and returns a best-effort string with `{???}` fallbacks.

---

## FluentResource Class

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/resource.ts

```typescript
class FluentResource {
  body: Array<Message | Term>;
  constructor(source: string);
}
```

Parses an FTL string into an optimized runtime representation. Uses a purpose-built parser that is smaller and faster than `@fluent/syntax` but does NOT produce a full AST.

- Parsing errors are per-message: a syntax error in one message does not prevent others from being parsed
- The parser minimizes false negatives at the expense of false positives
- For strict validation, use `@fluent/syntax` instead
- `FluentResource.fromString()` was removed in v0.14.0 -- ALWAYS use the constructor

---

## FluentType Hierarchy

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/types.ts

### FluentType\<T\> (abstract base)

```typescript
abstract class FluentType<T> {
  value: T;
  constructor(value: T);
  valueOf(): T;
  abstract toString(scope: Scope): string;
}
```

### FluentNone

```typescript
class FluentNone extends FluentType<string> {
  constructor(value?: string); // default: "???"
  toString(scope: Scope): string; // returns `{${this.value}}`
}
```

Represents a missing or failed value. Used as a fallback during resolution errors.

### FluentNumber

```typescript
class FluentNumber extends FluentType<number> {
  opts: Intl.NumberFormatOptions;
  constructor(value: number, opts?: Intl.NumberFormatOptions);
  toString(scope?: Scope): string;
}
```

Wraps a number with locale-aware formatting options. Delegates to `Intl.NumberFormat`.

### FluentDateTime

```typescript
class FluentDateTime extends FluentType<number> {
  opts: Intl.DateTimeFormatOptions;
  constructor(
    value: number | Date | TemporalObject | FluentDateTime | FluentType<number>,
    opts?: Intl.DateTimeFormatOptions
  );
  static supportsValue(value: unknown): boolean;
  toNumber(): number;
  toString(scope?: Scope): string;
}
```

Wraps a date/time value with locale-aware formatting options. Delegates to `Intl.DateTimeFormat`. Supports TC39 Temporal objects as of v0.19.0.

---

## Type Aliases

```typescript
// A resolved Fluent value
type FluentValue = FluentType<unknown> | string;

// Accepted variable types for formatPattern args
type FluentVariable = FluentValue | TemporalObject | string | number | Date;

// Custom function signature
type FluentFunction = (
  positional: Array<FluentValue>,
  named: Record<string, FluentValue>
) => FluentValue;

// Text transform applied to all TextElement strings
type TextTransform = (text: string) => string;
```

---

## Built-in Functions

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/builtins.ts

### NUMBER()

```typescript
function NUMBER(
  args: Array<FluentValue>,
  opts: Record<string, FluentValue>
): FluentValue
```

Formats numeric values using `Intl.NumberFormat`. Allowed options: `unitDisplay`, `currencyDisplay`, `useGrouping`, `minimumIntegerDigits`, `minimumFractionDigits`, `maximumFractionDigits`, `minimumSignificantDigits`, `maximumSignificantDigits`.

### DATETIME()

```typescript
function DATETIME(
  args: Array<FluentValue>,
  opts: Record<string, FluentValue>
): FluentValue
```

Formats date/time values using `Intl.DateTimeFormat`. Allowed options: `dateStyle`, `timeStyle`, `fractionalSecondDigits`, `dayPeriod`, `hour12`, `weekday`, `era`, `year`, `month`, `day`, `hour`, `minute`, `second`, `timeZoneName`.

Both functions return `FluentNone` if the input is `FluentNone` (propagating failures gracefully). They throw `TypeError` for invalid argument types.

---

## Temporal Support (v0.19.0+)

```typescript
interface TemporalInstant {
  epochMilliseconds: number;
  toString(): string;
}

interface TemporalDateTypes {
  calendarId: string;
  toZonedDateTime?(timeZone: string): { epochMilliseconds: number };
  toString(): string;
}

interface TemporalPlainTime {
  hour: number;
  minute: number;
  second: number;
  toString(): string;
}

type TemporalObject = TemporalInstant | TemporalDateTypes | TemporalPlainTime;
```

`FluentVariable` accepts Temporal objects directly -- no wrapping needed.

---

## Public Exports from @fluent/bundle

| Export | Kind | Purpose |
|--------|------|---------|
| `FluentBundle` | Class | Core API for formatting messages |
| `FluentResource` | Class | Parses FTL source into runtime representation |
| `FluentType` | Class | Abstract base for typed values |
| `FluentNone` | Class | Represents missing/failed values |
| `FluentNumber` | Class | Locale-aware number wrapper |
| `FluentDateTime` | Class | Locale-aware date/time wrapper |
| `FluentValue` | Type | `FluentType<unknown> \| string` |
| `FluentVariable` | Type | All accepted variable types for args |
| `FluentFunction` | Type | Custom function signature |
| `TextTransform` | Type | Text transformation function |
| `Message` | Type | `{ value: Pattern \| null; attributes: Record<string, Pattern> }` |
| `Scope` | Type | Internal resolution scope |
