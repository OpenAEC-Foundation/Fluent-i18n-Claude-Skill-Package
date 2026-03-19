# Part 2: @fluent/bundle API

> **Research date**: 2026-03-19
> **Sources**: All claims verified via WebFetch against official GitHub sources and npm registry.

---

## 2.1 Package Overview and Version

**Current version**: 0.19.1 (released April 2, 2025)
**npm package**: `@fluent/bundle`
**Repository**: https://github.com/projectfluent/fluent.js/tree/main/fluent-bundle
**License**: Apache-2.0 OR MIT
**Node.js requirement**: `^20.19 || ^22.12 || >=24`

The `@fluent/bundle` package is the core JavaScript runtime for Project Fluent. It provides the `FluentBundle` class that parses FTL resources and formats localized messages at runtime. The package was renamed from `fluent` to `@fluent/bundle` in version 0.13.0 (July 2019).

**Entry points** (from package.json):
- `main`: `./index.js` (CommonJS)
- `module`: `./esm/index.js` (ES modules)
- `types`: `./esm/index.d.ts` (TypeScript declarations)

**Runtime requirements**: The library requires the following `Intl` APIs to be available:
- `Intl.DateTimeFormat`
- `Intl.NumberFormat`
- `Intl.PluralRules`

**Public exports** (from `@fluent/bundle`):
- Classes: `FluentBundle`, `FluentResource`, `FluentType`, `FluentNone`, `FluentNumber`, `FluentDateTime`
- Types: `FluentValue`, `FluentVariable`, `FluentFunction`, `TextTransform`, `Message`, `Scope`

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/index.ts

---

## 2.2 FluentBundle Class

The `FluentBundle` class is the central API for formatting Fluent messages. It holds parsed resources, resolves message references, and formats patterns with arguments.

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/bundle.ts

### 2.2.1 Constructor and Options

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

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `locales` | `string \| Array<string>` | (required) | BCP 47 locale identifier(s) used by `Intl` formatters |
| `functions` | `Record<string, FluentFunction>` | `{}` | Custom functions available in FTL, merged with built-in NUMBER and DATETIME |
| `useIsolating` | `boolean` | `true` | Wrap placeables in Unicode isolation marks (U+2068/U+2069) for bidirectional text |
| `transform` | `TextTransform` | `(v) => v` | Transform applied to all TextElement strings in patterns |

Where `TextTransform` is:
```typescript
type TextTransform = (text: string) => string;
```

**Example — basic bundle creation**:
```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
```

**Example — bundle with custom functions and options**:
```typescript
import { FluentBundle, FluentResource, FluentVariable } from "@fluent/bundle";

const bundle = new FluentBundle("en-US", {
  useIsolating: false,
  functions: {
    UPCASE: (positional, named) => {
      const val = positional[0];
      return typeof val === "string" ? val.toUpperCase() : String(val).toUpperCase();
    },
  },
  transform: (text) => text, // identity transform (default)
});
```

**Internal properties** (public but prefixed with underscore — not part of stable API):
- `locales: Array<string>` — the resolved locale array
- `_terms: Map<string, Term>` — parsed terms (prefixed with `-` in FTL)
- `_messages: Map<string, Message>` — parsed messages
- `_functions: Record<string, FluentFunction>` — merged custom + built-in functions
- `_useIsolating: boolean`
- `_transform: TextTransform`
- `_intls: IntlCache` — memoized Intl formatter instances

### 2.2.2 addResource()

```typescript
addResource(
  res: FluentResource,
  options?: { allowOverrides?: boolean }
): Array<Error>
```

Adds a `FluentResource` to the bundle. Messages and terms from the resource become available for formatting.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `res` | `FluentResource` | (required) | A parsed FTL resource |
| `allowOverrides` | `boolean` | `false` | If `true`, new messages/terms overwrite existing ones with the same ID. If `false`, duplicates produce an error. |

**Return value**: An array of `Error` objects for any messages that had syntax errors during parsing. Syntax errors are **per-message** — they do not prevent other valid messages in the same resource from being added.

**Example**:
```typescript
const resource = new FluentResource(`
hello = Hello, world!
goodbye = Goodbye, {$name}!
`);

const errors = bundle.addResource(resource);
if (errors.length > 0) {
  console.error("FTL syntax errors:", errors);
}
```

**Example — allowing overrides**:
```typescript
const base = new FluentResource(`hello = Hello`);
const override = new FluentResource(`hello = Hi there`);

bundle.addResource(base);
bundle.addResource(override, { allowOverrides: true });
// "hello" now resolves to "Hi there"
```

### 2.2.3 getMessage()

```typescript
getMessage(id: string): Message | undefined
```

Retrieves a parsed message by its identifier. Returns `undefined` if the message does not exist. Terms (identifiers starting with `-`) are NOT accessible via `getMessage()`.

The returned `Message` object has this shape:
```typescript
interface Message {
  value: Pattern | null;
  attributes: Record<string, Pattern>;
}
```

- `value` — the main pattern of the message, or `null` if the message only has attributes
- `attributes` — a record of attribute patterns keyed by attribute name

**Example**:
```typescript
const resource = new FluentResource(`
login-input =
    .placeholder = Email address
    .aria-label = Login input
`);
bundle.addResource(resource);

const msg = bundle.getMessage("login-input");
// msg.value is null (message has no value, only attributes)
// msg.attributes["placeholder"] is a Pattern
// msg.attributes["aria-label"] is a Pattern

if (msg) {
  if (msg.value) {
    const text = bundle.formatPattern(msg.value);
  }
  const placeholder = bundle.formatPattern(msg.attributes["placeholder"]);
  // → "Email address"
}
```

### 2.2.4 hasMessage()

```typescript
hasMessage(id: string): boolean
```

Returns `true` if the bundle contains a message with the given identifier. Like `getMessage()`, this does NOT check for terms.

**Example**:
```typescript
if (bundle.hasMessage("welcome")) {
  const msg = bundle.getMessage("welcome")!;
  const text = bundle.formatPattern(msg.value!);
}
```

### 2.2.5 formatPattern()

```typescript
formatPattern(
  pattern: Pattern,
  args?: Record<string, FluentVariable> | null,
  errors?: Array<Error> | null
): string
```

The core formatting method. Takes a `Pattern` (obtained from `getMessage()`) and returns a fully resolved string.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `pattern` | `Pattern` | (required) | A pattern from `getMessage().value` or `getMessage().attributes[name]` |
| `args` | `Record<string, FluentVariable> \| null` | `null` | Variables to substitute into the pattern |
| `errors` | `Array<Error> \| null` | `null` | If provided, resolution errors are appended here instead of being thrown |

Where `FluentVariable` is:
```typescript
type FluentVariable = FluentValue | TemporalObject | string | number | Date;
```

And `FluentValue` is:
```typescript
type FluentValue = FluentType<unknown> | string;
```

**Critical behavior regarding errors**: If the `errors` array is NOT provided and a resolution error occurs, `formatPattern` **throws** on the first error. If `errors` IS provided, errors are collected and the formatter returns a best-effort string with `{???}` placeholders for failed parts.

**Example — basic formatting**:
```typescript
const resource = new FluentResource(`
welcome = Welcome, {$name}, to {-brand-name}!
-brand-name = Foo 3000
`);
bundle.addResource(resource);

const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "Anna" });
  // → "Welcome, Anna, to Foo 3000!"
}
```

**Example — with error collection**:
```typescript
const errors: Error[] = [];
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "Anna" }, errors);
  if (errors.length > 0) {
    console.warn("Formatting issues:", errors);
  }
}
```

**Example — formatting with numbers and dates**:
```typescript
const resource = new FluentResource(`
items-count = You have {$count} items.
last-login = Last login: {$date}
price = Total: {NUMBER($amount, minimumFractionDigits: 2)}
`);
bundle.addResource(resource);

const msg = bundle.getMessage("items-count");
if (msg?.value) {
  bundle.formatPattern(msg.value, { count: 5 });
  // → "You have 5 items."
}

const loginMsg = bundle.getMessage("last-login");
if (loginMsg?.value) {
  bundle.formatPattern(loginMsg.value, { date: new Date() });
  // → "Last login: 3/19/2026" (locale-dependent formatting)
}
```

---

## 2.3 FluentResource Class

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/resource.ts

```typescript
class FluentResource {
  body: Array<Message | Term>;
  constructor(source: string);
}
```

`FluentResource` parses an FTL string into an internal representation optimized for runtime use. It uses a purpose-built parser that is smaller and faster than the full `@fluent/syntax` parser but does NOT produce a full AST.

**Key characteristics**:
- The `body` property contains an array of parsed `Message` and `Term` objects
- Parsing errors are **per-message**: a syntax error in one message does not prevent other messages from being parsed
- The parser "minimizes false negatives at the expense of increasing false positives" — it accepts valid messages with near-perfect accuracy but may parse some technically invalid structures
- For strict validation, use `@fluent/syntax` instead

**Example**:
```typescript
import { FluentResource } from "@fluent/bundle";

const resource = new FluentResource(`
hello = Hello, world!
welcome = Welcome, {$name}!
-brand = Acme Corp
`);

console.log(resource.body.length); // 3 (2 messages + 1 term)
```

**Version history note**: Before v0.14.0, `FluentResource` extended `Map` and had a static `fromString()` method. Both were removed. ALWAYS use the constructor directly.

---

## 2.4 Error Handling

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/resolver.ts

Fluent's error handling philosophy is **resilience over failure**. The resolver attempts to salvage as much of the translation as possible, replacing failed parts with `{???}` fallback values.

### Error Types During Resolution

| Error Type | Condition | Fallback |
|-----------|-----------|----------|
| `ReferenceError` | Unknown variable: `$name` | `{$name}` |
| `ReferenceError` | Unknown message: `msg-id` | `{msg-id}` |
| `ReferenceError` | Unknown term: `-term-id` | `{-term-id}` |
| `ReferenceError` | Unknown attribute: `msg.attr` | `{msg.attr}` |
| `ReferenceError` | No value on message (only attributes exist) | `{msg-id}` |
| `ReferenceError` | Unknown function: `FUNC()` | `{FUNC()}` |
| `TypeError` | Unsupported variable type passed as argument | `{$name}` |
| `TypeError` | Function is not callable | `{FUNC()}` |
| `RangeError` | Cyclic reference detected | `{???}` |
| `RangeError` | No default variant in select expression | `{???}` |
| `RangeError` | Excessive placeables (>100) — **fatal, terminates resolution** | Throws |

### Two Error Handling Modes

**Mode 1 — Silent collection** (recommended for production):
```typescript
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
// errors array now contains any issues
// text contains best-effort output with {???} for failed parts
```

**Mode 2 — Throw on first error** (useful for development/testing):
```typescript
// No errors array → throws on first resolution error
const text = bundle.formatPattern(pattern, args);
// Throws ReferenceError, TypeError, or RangeError
```

### Parse-Time vs. Resolution-Time Errors

- **Parse-time errors** (returned by `addResource()`): Syntax errors in FTL. These are per-message — one bad message does not break others.
- **Resolution-time errors** (from `formatPattern()`): Missing variables, broken references, type mismatches. These occur when formatting a specific pattern.

---

## 2.5 Custom Functions

Custom functions extend Fluent's formatting capabilities beyond the built-in `NUMBER()` and `DATETIME()`.

### FluentFunction Type Signature

```typescript
type FluentFunction = (
  positional: Array<FluentValue>,
  named: Record<string, FluentValue>
) => FluentValue;
```

Where `FluentValue = FluentType<unknown> | string`.

Functions receive two arguments:
1. `positional` — positional arguments from the FTL call expression
2. `named` — named arguments as key-value pairs

### Registering Custom Functions

```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    GREETING_TIME: (positional, named) => {
      const hour = new Date().getHours();
      if (hour < 12) return "morning";
      if (hour < 18) return "afternoon";
      return "evening";
    },
    SHOUT: (positional, named) => {
      const text = positional[0];
      return typeof text === "string" ? text.toUpperCase() : String(text).toUpperCase();
    },
  },
});
```

Corresponding FTL:
```ftl
greeting = Good {GREETING_TIME()}!
shout = {SHOUT("hello")}
```

### Built-in Functions: NUMBER and DATETIME

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/builtins.ts

**NUMBER()** — formats numeric values using `Intl.NumberFormat`.

```typescript
function NUMBER(
  args: Array<FluentValue>,
  opts: Record<string, FluentValue>
): FluentValue
```

Allowed options (passed through to `Intl.NumberFormat`):
`unitDisplay`, `currencyDisplay`, `useGrouping`, `minimumIntegerDigits`, `minimumFractionDigits`, `maximumFractionDigits`, `minimumSignificantDigits`, `maximumSignificantDigits`

FTL usage:
```ftl
price = { NUMBER($amount, minimumFractionDigits: 2) }
```

**DATETIME()** — formats date/time values using `Intl.DateTimeFormat`.

```typescript
function DATETIME(
  args: Array<FluentValue>,
  opts: Record<string, FluentValue>
): FluentValue
```

Allowed options (passed through to `Intl.DateTimeFormat`):
`dateStyle`, `timeStyle`, `fractionalSecondDigits`, `dayPeriod`, `hour12`, `weekday`, `era`, `year`, `month`, `day`, `hour`, `minute`, `second`, `timeZoneName`

FTL usage:
```ftl
last-seen = Last seen: { DATETIME($date, dateStyle: "long") }
```

Both functions return `FluentNone` if the input argument is `FluentNone`, effectively propagating failures gracefully. They throw `TypeError` for completely invalid argument types.

---

## 2.6 @fluent/syntax Package (Parser/Serializer)

**Current version**: 0.19.0
**npm package**: `@fluent/syntax`
**Node.js requirement**: `^20.19 || ^22.12 || >=24`

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-syntax/README.md

The `@fluent/syntax` package provides a **full AST parser and serializer** for Fluent files. It is a separate package from `@fluent/bundle` and serves a different purpose.

### When to Use Which

| Use Case | Package |
|----------|---------|
| Runtime message formatting in apps | `@fluent/bundle` |
| Linting/validating FTL files | `@fluent/syntax` |
| Programmatic FTL generation or modification | `@fluent/syntax` |
| Building editor tooling (syntax highlighting, etc.) | `@fluent/syntax` |
| Extracting message IDs from FTL files | `@fluent/syntax` |
| Converting between formats | `@fluent/syntax` |

### FluentParser

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-syntax/src/parser.ts

```typescript
interface FluentParserOptions {
  withSpans?: boolean; // default: true
}

class FluentParser {
  constructor(options?: FluentParserOptions);
  parse(source: string): Resource;
  parseEntry(source: string): Entry;
}
```

- `parse(source)` — parses a complete FTL document into a `Resource` AST node
- `parseEntry(source)` — parses the first Message or Term from the input; returns `Junk` on failure
- `withSpans: true` — each AST node includes a `Span` with `start` and `end` byte offsets

**Example**:
```typescript
import { FluentParser } from "@fluent/syntax";

const parser = new FluentParser({ withSpans: true });
const resource = parser.parse(`
hello = Hello, world!
welcome = Welcome, {$name}!
`);

for (const entry of resource.body) {
  if (entry.type === "Message") {
    console.log(entry.id.name); // "hello", "welcome"
    console.log(entry.span?.start, entry.span?.end);
  }
}
```

### FluentSerializer

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-syntax/src/serializer.ts

```typescript
interface FluentSerializerOptions {
  withJunk?: boolean; // default: false
}

class FluentSerializer {
  withJunk: boolean;
  constructor(options?: FluentSerializerOptions);
  serialize(resource: Resource): string;
  serializeEntry(entry: Entry, state?: number): string;
}
```

Additionally exported standalone functions:
```typescript
function serializeExpression(expr: Expression | Placeable): string;
function serializeVariantKey(key: Identifier | NumberLiteral): string;
```

**Example — round-trip parse and serialize**:
```typescript
import { FluentParser, FluentSerializer } from "@fluent/syntax";

const parser = new FluentParser();
const serializer = new FluentSerializer();

const resource = parser.parse(`hello = Hello, world!`);
const output = serializer.serialize(resource);
// → "hello = Hello, world!\n"
```

### AST Types

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-syntax/src/ast.ts

The full AST hierarchy:

**Base**: `BaseNode` → `SyntaxNode` (adds optional `span: Span`)

**Top-level**:
- `Resource` — `body: Array<Entry>` where `Entry = Message | Term | Comment | GroupComment | ResourceComment | Junk`

**Messages and Terms**:
- `Message` — `id: Identifier`, `value: Pattern | null`, `attributes: Array<Attribute>`, `comment: Comment | null`
- `Term` — `id: Identifier`, `value: Pattern`, `attributes: Array<Attribute>`, `comment: Comment | null`
- `Attribute` — `id: Identifier`, `value: Pattern`

**Patterns**:
- `Pattern` — `elements: Array<PatternElement>` where `PatternElement = TextElement | Placeable`
- `TextElement` — `value: string`
- `Placeable` — `expression: Expression`

**Expressions**:
- `StringLiteral` — `value: string` (with `parse()` method for escape sequences)
- `NumberLiteral` — `value: string` (with `parse()` method)
- `MessageReference` — `id: Identifier`, `attribute: Identifier | null`
- `TermReference` — `id: Identifier`, `attribute: Identifier | null`, `arguments: CallArguments | null`
- `VariableReference` — `id: Identifier`
- `FunctionReference` — `id: Identifier`, `arguments: CallArguments`
- `SelectExpression` — `selector: InlineExpression`, `variants: Array<Variant>`

**Supporting**:
- `Variant` — `key: Identifier | NumberLiteral`, `value: Pattern`, `default: boolean`
- `CallArguments` — `positional: Array<InlineExpression>`, `named: Array<NamedArgument>`
- `NamedArgument` — `name: Identifier`, `value: Literal`
- `Identifier` — `name: string`

**Comments**:
- `Comment` — `content: string` (single `#`)
- `GroupComment` — `content: string` (double `##`)
- `ResourceComment` — `content: string` (triple `###`)

**Errors**:
- `Junk` — `content: string`, `annotations: Array<Annotation>`
- `Annotation` — `code: string`, `arguments: Array<unknown>`, `message: string`
- `Span` — `start: number`, `end: number`

---

## 2.7 Type Exports and TypeScript Usage

The `@fluent/bundle` package is written in TypeScript (migrated in v0.15.0, January 2020). Type declarations are published alongside the ES module build at `esm/index.d.ts`.

### Key Types for Consumers

```typescript
import type {
  FluentValue,      // FluentType<unknown> | string
  FluentVariable,   // FluentValue | TemporalObject | string | number | Date
  FluentFunction,   // (positional: FluentValue[], named: Record<string, FluentValue>) => FluentValue
  TextTransform,    // (text: string) => string
  Message,          // { value: Pattern | null; attributes: Record<string, Pattern> }
} from "@fluent/bundle";
```

### FluentType Hierarchy

```typescript
abstract class FluentType<T> {
  value: T;
  constructor(value: T);
  valueOf(): T;
  abstract toString(scope: Scope): string;
}

class FluentNone extends FluentType<string> {
  constructor(value?: string); // default: "???"
  toString(scope: Scope): string; // returns `{${this.value}}`
}

class FluentNumber extends FluentType<number> {
  opts: Intl.NumberFormatOptions;
  constructor(value: number, opts?: Intl.NumberFormatOptions);
  toString(scope?: Scope): string;
}

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

### Temporal Support (v0.19.0+)

As of version 0.19.0, `FluentDateTime` supports TC39 Temporal objects:

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

`FluentVariable` accepts Temporal objects directly:
```typescript
type FluentVariable = FluentValue | TemporalObject | string | number | Date;
```

---

## 2.8 Version History and Breaking Changes

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/CHANGELOG.md

### Major Version Milestones

| Version | Date | Key Change |
|---------|------|------------|
| 0.19.1 | 2025-04-02 | Fix FluentDateTime/FluentNumber primitive conversions |
| 0.19.0 | 2025-03-25 | Temporal support; Node.js >= 18 required |
| 0.18.0 | 2023-03-13 | Drop Node.js v12; documentation format update |
| 0.17.0 | 2021-09-13 | ESM-first: `"type": "module"` in esm/ directory; Node.js 12 minimum |
| 0.16.0 | 2020-07-01 | Rename `FluentArgument` → `FluentVariable`; remove compat builds; ES2018 target |
| 0.15.0 | 2020-01-23 | Codebase migrated to TypeScript; `FluentValue` type exported |
| 0.14.0 | 2019-07-30 | **Major API overhaul**: remove `addMessages`, `format`; add `formatPattern`, `FluentResource` constructor |
| 0.13.0 | 2019-07-25 | Package renamed from `fluent` to `@fluent/bundle` |
| 0.8.0 | 2018-08-20 | `MessageContext` → `FluentBundle`; `MessageArgument` → `FluentType` |

### Critical Breaking Changes to Know

**v0.14.0 (the big API break)**:
- `FluentBundle.addMessages(source)` removed → use `addResource(new FluentResource(source))`
- `FluentBundle.format(msg, args, errors)` removed → use `formatPattern(getMessage(id).value, args, errors)`
- `FluentBundle.messages` property removed → use `getMessage()` and `hasMessage()`
- `FluentResource.fromString()` removed → use `new FluentResource(source)`
- `getMessage()` return shape changed to `{ value: Pattern | null, attributes: Record<string, Pattern> }`
- `formatPattern` **throws** if no errors array is provided (unlike old `format` which silently returned)

**v0.16.0**:
- `FluentArgument` type renamed to `FluentVariable`
- Compat builds removed — everything compiles to ES2018

**v0.15.1**:
- NUMBER and DATETIME builtins now accept ONLY specific formatting options (see Section 2.5 for allowed lists)

**v0.19.0**:
- Requires Node.js >= 18

---

## 2.9 Common Issues from GitHub

Source: https://github.com/projectfluent/fluent.js/issues

### Open Issues (as of 2026-03-19)

| # | Title | Category | Impact |
|---|-------|----------|--------|
| 648 | Using Fluent with a lot of roots is slow | Performance | Scale issues when many FluentBundle instances are created |
| 646 | Usage example for SSR on Next.js | Documentation | No official SSR guidance |
| 628 | How can a variable be referenced in terms params? | API confusion | Terms parameterization not intuitive |
| 626 | Synchronising .ftl files and finding fluent IDs | Tooling | No built-in sync tooling |
| 625 | RFC: fluent babel plugin | Feature request | No build-time optimization |
| 624 | Support prefix identifier for message | Feature request | Namespace management |
| 615 | DOMLocalization setArgs method | Enhancement | DOM API limitations |
| 599 | Disabling unicode isolation per-placeable | Feature request | Fine-grained bidi control |
| 598 | Support ref forwarding in withLocalization | React | React pattern incompatibility |
| 597 | Support range versions of NUMBER/DATETIME | Feature request | Limited built-in formatting |
| 587 | Transform placeables (escape values) | Feature request | No placeable-level transform |
| 583 | Automatic Eastern Arabic numerals | Localization | Incomplete locale formatting |

### Key Closed Issues

| # | Title | Impact |
|---|-------|--------|
| 376 | Migrate to TypeScript? | Resolved in v0.15.0 |
| 217 | Introduce FluentResource and addResource | Resolved in v0.14.0 |
| 208 | format() should accept a string identifier | Led to getMessage + formatPattern two-step API |
| 364 | Emit errors on FluentBundle instances | Led to current error array pattern |
| 222 | Rename MessageContext to FluentContext | Resolved as FluentBundle in v0.8.0 |

### Patterns in Issues

1. **Performance at scale**: Multiple bundles (one per locale per component) creates overhead. No built-in bundle-sharing strategy.
2. **SSR gaps**: No official guidance for server-side rendering with Fluent.
3. **Terms confusion**: How terms interact with parameterization is a common source of questions.
4. **React integration friction**: `withLocalization` HOC does not support ref forwarding, pushing users toward hooks.
5. **Bidi control**: `useIsolating: true` is all-or-nothing; users want per-placeable control.

---

## 2.10 Anti-Patterns

Based on API source code, GitHub issues, and changelog analysis, the following anti-patterns MUST be avoided:

### Anti-Pattern 1: Passing FTL strings directly to addResource

```typescript
// WRONG — addResource expects a FluentResource, not a string
bundle.addResource(`hello = Hello`); // TypeError

// CORRECT
bundle.addResource(new FluentResource(`hello = Hello`));
```

This mistake is common among developers migrating from the pre-0.14.0 API where `addMessages()` accepted raw strings.

### Anti-Pattern 2: Ignoring the two-step getMessage + formatPattern flow

```typescript
// WRONG — there is no bundle.format() method (removed in v0.14.0)
bundle.format("welcome", { name: "Anna" }); // TypeError

// CORRECT — two-step: get message, then format its pattern
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  bundle.formatPattern(msg.value, { name: "Anna" });
}
```

### Anti-Pattern 3: Not checking for null value

```typescript
// DANGEROUS — messages can have null values (attributes-only messages)
const msg = bundle.getMessage("login-input")!;
bundle.formatPattern(msg.value!, args); // Runtime error if value is null

// CORRECT — always check
const msg = bundle.getMessage("login-input");
if (msg?.value) {
  bundle.formatPattern(msg.value, args);
}
```

### Anti-Pattern 4: Omitting the errors array in production

```typescript
// DANGEROUS — throws on first resolution error, crashing the app
const text = bundle.formatPattern(pattern, args);

// CORRECT for production — collect errors gracefully
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
```

### Anti-Pattern 5: Trying to access terms via getMessage

```typescript
// WRONG — terms (prefixed with -) are not accessible via getMessage
const term = bundle.getMessage("-brand-name"); // returns undefined

// Terms are resolved automatically when referenced in messages:
// welcome = Welcome to {-brand-name}!
// There is NO public API to directly format a term.
```

### Anti-Pattern 6: Creating FluentResource per message

```typescript
// WRONG — inefficient, creates parser overhead per message
for (const [id, text] of Object.entries(messages)) {
  bundle.addResource(new FluentResource(`${id} = ${text}`));
}

// CORRECT — batch all messages into one resource
const ftl = Object.entries(messages)
  .map(([id, text]) => `${id} = ${text}`)
  .join("\n");
bundle.addResource(new FluentResource(ftl));
```

### Anti-Pattern 7: Using @fluent/syntax for runtime formatting

```typescript
// WRONG — @fluent/syntax is a tooling library, too heavy for runtime
import { FluentParser } from "@fluent/syntax";
const ast = new FluentParser().parse(ftlSource);
// ...manually walking the AST to format messages

// CORRECT — use @fluent/bundle for runtime
import { FluentBundle, FluentResource } from "@fluent/bundle";
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftlSource));
```

`@fluent/bundle` includes its own optimized runtime parser. `@fluent/syntax` is for tooling: linting, programmatic FTL generation, editor integrations.

### Anti-Pattern 8: Passing unsupported types as variables

```typescript
// WRONG — objects, arrays, booleans are not valid FluentVariable types
bundle.formatPattern(pattern, { user: { name: "Anna" } }); // TypeError

// CORRECT — only string, number, Date, FluentType, or Temporal objects
bundle.formatPattern(pattern, { name: "Anna", count: 5, date: new Date() });
```

### Anti-Pattern 9: Not handling addResource errors

```typescript
// WRONG — silently ignoring parse errors
bundle.addResource(new FluentResource(userProvidedFtl));

// CORRECT — check and log errors
const errors = bundle.addResource(new FluentResource(userProvidedFtl));
if (errors.length > 0) {
  errors.forEach(e => console.error("FTL parse error:", e.message));
}
```

### Anti-Pattern 10: Disabling useIsolating without understanding bidi

```typescript
// RISKY — disabling isolation marks can break RTL/LTR mixed text
const bundle = new FluentBundle("ar", { useIsolating: false });

// ONLY disable when you are certain your content is unidirectional
// or when testing and the Unicode marks interfere with assertions
const bundle = new FluentBundle("en-US", { useIsolating: false });
```
