# Key Types Reference

## FluentBundle

The central API class. Holds parsed resources for a specific locale and formats messages.

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
| `locales` | `string \| Array<string>` | (required) | BCP 47 locale identifier(s) for `Intl` formatters |
| `functions` | `Record<string, FluentFunction>` | `{}` | Custom functions merged with built-in NUMBER and DATETIME |
| `useIsolating` | `boolean` | `true` | Wrap placeables in Unicode bidi isolation marks (U+2068/U+2069) |
| `transform` | `TextTransform` | `(v) => v` | Transform applied to all TextElement strings |

### Methods

#### addResource

```typescript
addResource(
  res: FluentResource,
  options?: { allowOverrides?: boolean }
): Array<Error>
```

Adds a `FluentResource` to the bundle. Returns an array of `Error` objects for any messages with syntax errors. Parse errors are per-message -- one bad message does NOT prevent others from being added.

- `allowOverrides: false` (default): duplicate message IDs produce an error
- `allowOverrides: true`: new messages overwrite existing ones with the same ID

#### getMessage

```typescript
getMessage(id: string): Message | undefined
```

Retrieves a parsed message by identifier. Returns `undefined` if the message does not exist. Terms (identifiers starting with `-`) are NEVER accessible via `getMessage()`.

#### hasMessage

```typescript
hasMessage(id: string): boolean
```

Returns `true` if the bundle contains a message with the given identifier. Does NOT check for terms.

#### formatPattern

```typescript
formatPattern(
  pattern: Pattern,
  args?: Record<string, FluentVariable> | null,
  errors?: Array<Error> | null
): string
```

Formats a `Pattern` (from `getMessage().value` or `getMessage().attributes[name]`) into a resolved string.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `pattern` | `Pattern` | (required) | A pattern obtained from `getMessage()` |
| `args` | `Record<string, FluentVariable> \| null` | `null` | Variables to substitute |
| `errors` | `Array<Error> \| null` | `null` | If provided, errors are collected; if omitted, throws on first error |

---

## FluentResource

Parses an FTL string into an internal representation optimized for runtime use.

```typescript
class FluentResource {
  body: Array<Message | Term>;
  constructor(source: string);
}
```

- Uses a purpose-built parser that is smaller and faster than `@fluent/syntax`
- Does NOT produce a full AST
- Parsing errors are per-message: one syntax error does not break other messages
- For strict validation, use `@fluent/syntax` instead

---

## Message

The shape returned by `FluentBundle.getMessage()`:

```typescript
interface Message {
  value: Pattern | null;
  attributes: Record<string, Pattern>;
}
```

- `value` -- the main pattern of the message, or `null` if the message has only attributes
- `attributes` -- a record of attribute patterns keyed by attribute name

ALWAYS check `msg.value` before formatting. Messages with only attributes have `value: null`.

---

## Pattern

An opaque type representing the parsed content of a message value or attribute. Obtained from `Message.value` or `Message.attributes[name]`. Passed to `formatPattern()` for resolution.

You NEVER construct a `Pattern` directly. It is always obtained from `getMessage()`.

---

## FluentVariable

The type accepted as variable values in `formatPattern()`:

```typescript
type FluentVariable = FluentValue | TemporalObject | string | number | Date;
```

Where:

```typescript
type FluentValue = FluentType<unknown> | string;
```

ALWAYS pass only `string`, `number`, `Date`, `FluentType` subclasses, or Temporal objects (v0.19.0+). Objects, arrays, and booleans are NOT valid and will cause a `TypeError`.

---

## FluentType Hierarchy

```typescript
abstract class FluentType<T> {
  value: T;
  valueOf(): T;
  abstract toString(scope: Scope): string;
}

class FluentNone extends FluentType<string> {
  // Default value: "???"
  // toString returns `{${this.value}}`
}

class FluentNumber extends FluentType<number> {
  opts: Intl.NumberFormatOptions;
  // Formats via Intl.NumberFormat
}

class FluentDateTime extends FluentType<number> {
  opts: Intl.DateTimeFormatOptions;
  // Formats via Intl.DateTimeFormat
  // Supports Temporal objects (v0.19.0+)
}
```

---

## FluentFunction

The signature for custom functions registered on a bundle:

```typescript
type FluentFunction = (
  positional: Array<FluentValue>,
  named: Record<string, FluentValue>
) => FluentValue;
```

### Built-in Functions

| Function | Purpose | Underlying API |
|----------|---------|---------------|
| `NUMBER()` | Format numbers with locale-aware rules | `Intl.NumberFormat` |
| `DATETIME()` | Format dates/times with locale-aware rules | `Intl.DateTimeFormat` |

Custom functions are registered via the `functions` option in the `FluentBundle` constructor. They are merged with the built-in functions.

---

## TextTransform

```typescript
type TextTransform = (text: string) => string;
```

Applied to all `TextElement` strings during pattern resolution. Default is the identity function. Use this for sanitization or text transformations that should apply to all static text in patterns.

---

## Intl Requirements

`@fluent/bundle` requires these `Intl` APIs at runtime:

- `Intl.DateTimeFormat`
- `Intl.NumberFormat`
- `Intl.PluralRules`

These are available in all modern browsers and Node.js 12+.
