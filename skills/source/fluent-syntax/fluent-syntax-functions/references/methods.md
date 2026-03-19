# Methods Reference -- fluent-syntax-functions

## NUMBER() Options (Intl.NumberFormat)

All options are passed through to `Intl.NumberFormat`. Use them as named arguments in FTL.

### Allowed Options in @fluent/bundle

| Option | Type | Description | FTL Example |
|--------|------|-------------|-------------|
| `minimumIntegerDigits` | `number` (1-21) | Minimum number of integer digits | `NUMBER($n, minimumIntegerDigits: 3)` → `005` |
| `minimumFractionDigits` | `number` (0-20) | Minimum number of fraction digits | `NUMBER($n, minimumFractionDigits: 2)` → `3.00` |
| `maximumFractionDigits` | `number` (0-20) | Maximum number of fraction digits | `NUMBER($n, maximumFractionDigits: 0)` → `12,346` |
| `minimumSignificantDigits` | `number` (1-21) | Minimum number of significant digits | `NUMBER($n, minimumSignificantDigits: 3)` → `3.00` |
| `maximumSignificantDigits` | `number` (1-21) | Maximum number of significant digits | `NUMBER($n, maximumSignificantDigits: 2)` → `12,000` |
| `useGrouping` | `number` (0 or 1) | Use grouping separators (thousands) | `NUMBER($n, useGrouping: 0)` → `12345` |
| `currencyDisplay` | `string` | How to display currency (`"symbol"`, `"code"`, `"name"`) | Used with custom currency functions |
| `unitDisplay` | `string` | How to display unit (`"short"`, `"long"`, `"narrow"`) | Used with custom unit functions |

### NUMBER() in Selectors vs Placeables

| Context | Purpose | Example |
|---------|---------|---------|
| Placeable | Display formatting | `{ NUMBER($count, minimumFractionDigits: 2) }` |
| Selector | Plural category | `{ NUMBER($count, type: "ordinal") -> ... }` |
| Both | Format + select | Format in selector AND repeat in variant body |

### Special Selector Option

| Option | Value | Purpose |
|--------|-------|---------|
| `type` | `"ordinal"` | Switches from cardinal to ordinal plural rules |

This option is NOT an `Intl.NumberFormat` option -- it is handled by the Fluent resolver to determine which `Intl.PluralRules` type to use for category matching.

---

## DATETIME() Options (Intl.DateTimeFormat)

All options are passed through to `Intl.DateTimeFormat`. Use them as named arguments in FTL.

### Allowed Options in @fluent/bundle

| Option | Accepted Values | Description | FTL Example |
|--------|----------------|-------------|-------------|
| `dateStyle` | `"full"`, `"long"`, `"medium"`, `"short"` | Preset date format | `DATETIME($d, dateStyle: "long")` |
| `timeStyle` | `"full"`, `"long"`, `"medium"`, `"short"` | Preset time format | `DATETIME($d, timeStyle: "short")` |
| `weekday` | `"narrow"`, `"short"`, `"long"` | Weekday representation | `DATETIME($d, weekday: "long")` |
| `era` | `"narrow"`, `"short"`, `"long"` | Era representation | `DATETIME($d, era: "long")` |
| `year` | `"numeric"`, `"2-digit"` | Year representation | `DATETIME($d, year: "numeric")` |
| `month` | `"numeric"`, `"2-digit"`, `"narrow"`, `"short"`, `"long"` | Month representation | `DATETIME($d, month: "long")` |
| `day` | `"numeric"`, `"2-digit"` | Day representation | `DATETIME($d, day: "numeric")` |
| `hour` | `"numeric"`, `"2-digit"` | Hour representation | `DATETIME($d, hour: "numeric")` |
| `minute` | `"numeric"`, `"2-digit"` | Minute representation | `DATETIME($d, minute: "2-digit")` |
| `second` | `"numeric"`, `"2-digit"` | Second representation | `DATETIME($d, second: "2-digit")` |
| `timeZoneName` | `"short"`, `"long"` | Time zone name display | `DATETIME($d, timeZoneName: "short")` |
| `hour12` | `number` (0 or 1) | Force 12/24 hour format | `DATETIME($d, hour12: 1)` |
| `dayPeriod` | `"narrow"`, `"short"`, `"long"` | Day period (AM/PM style) | `DATETIME($d, dayPeriod: "short")` |
| `fractionalSecondDigits` | `number` (1-3) | Sub-second precision | `DATETIME($d, fractionalSecondDigits: 3)` |

### dateStyle/timeStyle vs Individual Options

**NEVER** combine `dateStyle`/`timeStyle` with individual date/time component options (`weekday`, `year`, `month`, `day`, `hour`, `minute`, `second`). This is an `Intl.DateTimeFormat` restriction -- mixing them throws a `TypeError`.

```ftl
# CORRECT -- use style shortcuts
event-date = { DATETIME($date, dateStyle: "long", timeStyle: "short") }

# CORRECT -- use individual options
event-date = { DATETIME($date, year: "numeric", month: "long", day: "numeric") }

# WRONG -- mixing styles and individual options
event-date = { DATETIME($date, dateStyle: "long", hour: "numeric") }
```

---

## FluentFunction Signature (TypeScript)

```typescript
type FluentFunction = (
  positional: Array<FluentValue>,
  named: Record<string, FluentValue>
) => FluentValue;

type FluentValue = FluentType<unknown> | string;
```

### Parameter Details

| Parameter | Type | Content |
|-----------|------|---------|
| `positional` | `Array<FluentValue>` | Positional args from FTL call: `FUNC($var, "literal")` → `[FluentNumber, "literal"]` |
| `named` | `Record<string, FluentValue>` | Named args from FTL call: `FUNC($var, opt: "val")` → `{ opt: "val" }` |

### Return Value

Functions MUST return a `FluentValue`:
- Return a `string` for simple text output
- Return a `FluentNumber` for numeric values that should participate in plural selection
- Return a `FluentDateTime` for date values

### How Arguments Arrive

| FTL Call | `positional` | `named` |
|----------|-------------|---------|
| `FUNC()` | `[]` | `{}` |
| `FUNC($var)` | `[FluentNumber(5)]` | `{}` |
| `FUNC("text")` | `["text"]` | `{}` |
| `FUNC($var, min: 2)` | `[FluentNumber(5)]` | `{ min: FluentNumber(2) }` |
| `FUNC("a", "b", x: 1)` | `["a", "b"]` | `{ x: FluentNumber(1) }` |

---

## FunctionReference Grammar (EBNF)

```ebnf
FunctionReference ::= Identifier CallArguments
CallArguments     ::= blank? "(" blank? argument_list blank? ")"
argument_list     ::= (Argument blank? "," blank?)* Argument?
Argument          ::= NamedArgument | InlineExpression
NamedArgument     ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
Identifier        ::= [a-zA-Z] [a-zA-Z0-9_-]*
```

### Grammar Constraints

1. Function names use the `Identifier` production -- uppercase is convention, not grammar-enforced
2. Named argument values are restricted to `StringLiteral` or `NumberLiteral` -- variable references (`$var`) are NOT allowed as named argument values
3. Positional arguments (`InlineExpression`) can be variables, literals, other function calls, or message/term references
4. Positional arguments MUST come before named arguments in the argument list
