# Anti-Patterns Reference -- fluent-syntax-functions

## Anti-Pattern 1: Lowercase Function Names in FTL

```ftl
# WRONG -- lowercase identifiers are parsed as message references, not functions
price = { number($amount, minimumFractionDigits: 2) }
date = { datetime($date, dateStyle: "long") }

# CORRECT -- ALWAYS use UPPERCASE for function names
price = { NUMBER($amount, minimumFractionDigits: 2) }
date = { DATETIME($date, dateStyle: "long") }
```

**Why**: The Fluent parser treats lowercase identifiers followed by parentheses as message references with call arguments (which is invalid for messages). ALWAYS use UPPERCASE for function names. This is a universal convention enforced by all Fluent tooling.

---

## Anti-Pattern 2: Passing Objects, Arrays, or Booleans as Variables

```typescript
// WRONG -- objects are not valid FluentVariable
bundle.formatPattern(pattern, {
  user: { name: "Anna", age: 30 },  // TypeError
});

// WRONG -- arrays are not valid FluentVariable
bundle.formatPattern(pattern, {
  items: ["apple", "banana"],  // TypeError
});

// WRONG -- booleans are not valid FluentVariable
bundle.formatPattern(pattern, {
  isAdmin: true,  // TypeError
});

// CORRECT -- only string, number, Date, FluentType, or Temporal
bundle.formatPattern(pattern, {
  userName: "Anna",           // string ✓
  userAge: 30,                // number ✓
  loginDate: new Date(),      // Date ✓
  isAdmin: "true",            // string representation ✓
  itemCount: 2,               // number ✓
});
```

**Why**: `FluentVariable` only accepts `string | number | Date | FluentType<unknown> | TemporalObject`. Passing any other type causes a `TypeError` at resolution time. For booleans, convert to string (`"true"`/`"false"`) and use string selectors. For objects, extract the needed fields as individual variables.

---

## Anti-Pattern 3: Variable References as Named Argument Values

```ftl
# WRONG -- named argument values MUST be literals, not variable references
price = { NUMBER($amount, minimumFractionDigits: $digits) }
date = { DATETIME($date, dateStyle: $style) }

# CORRECT -- use literal values for named arguments
price = { NUMBER($amount, minimumFractionDigits: 2) }
date = { DATETIME($date, dateStyle: "long") }
```

**Why**: The FTL grammar restricts named argument values to `StringLiteral | NumberLiteral`. Variable references (`$var`) are NOT allowed as named argument values. This is a parser-level restriction, not a runtime limitation.

---

## Anti-Pattern 4: Disabling useIsolating for RTL/LTR Mixed Content

```typescript
// WRONG -- disabling isolation for Arabic content with LTR placeables
const bundle = new FluentBundle("ar", { useIsolating: false });
bundle.addResource(new FluentResource(`
file-info = الملف { $filename } بحجم { $size } بايت
`));

// The LTR filename "report.pdf" will corrupt the surrounding RTL text layout
// Numbers and punctuation may appear in unexpected positions
```

```typescript
// CORRECT -- keep default useIsolating: true for mixed directionality
const bundle = new FluentBundle("ar"); // useIsolating defaults to true
bundle.addResource(new FluentResource(`
file-info = الملف { $filename } بحجم { $size } بايت
`));

// Unicode isolation marks (U+2068/U+2069) properly contain LTR placeables
```

**Why**: Unicode bidirectional algorithm can produce unexpected results when LTR text (Latin filenames, numbers) is embedded in RTL text (Arabic, Hebrew) without isolation. The default `useIsolating: true` wraps every placeable in First Strong Isolate (U+2068) and Pop Directional Isolate (U+2069) marks. ONLY disable this for testing or when ALL content is guaranteed unidirectional.

---

## Anti-Pattern 5: Mixing dateStyle/timeStyle with Individual Components

```ftl
# WRONG -- dateStyle/timeStyle cannot be combined with individual options
event = { DATETIME($date, dateStyle: "long", hour: "numeric") }
meeting = { DATETIME($date, timeStyle: "short", year: "numeric") }

# CORRECT -- use EITHER style shortcuts OR individual options
event = { DATETIME($date, dateStyle: "long", timeStyle: "short") }
meeting = { DATETIME($date, year: "numeric", month: "long", day: "numeric", hour: "numeric", minute: "2-digit") }
```

**Why**: `Intl.DateTimeFormat` throws a `TypeError` when `dateStyle` or `timeStyle` is combined with individual component options (`weekday`, `year`, `month`, `day`, `hour`, `minute`, `second`). This is a JavaScript runtime restriction, not a Fluent limitation. Choose one approach or the other.

---

## Anti-Pattern 6: Overriding Built-in NUMBER/DATETIME Without Intent

```typescript
// WRONG -- accidentally overriding built-in NUMBER
const bundle = new FluentBundle("en-US", {
  functions: {
    NUMBER: (positional, named) => {
      // This REPLACES the built-in NUMBER function entirely
      return String(positional[0]);
    },
  },
});
// All NUMBER() calls in FTL now use this custom function
// Plural rules, locale formatting, and options like minimumFractionDigits stop working
```

```typescript
// CORRECT -- use a different name for custom number formatting
const bundle = new FluentBundle("en-US", {
  functions: {
    CUSTOM_NUMBER: (positional, named) => {
      return String(positional[0]);
    },
  },
});
```

**Why**: Custom functions are merged with built-ins. If you name a custom function `NUMBER` or `DATETIME`, it completely replaces the built-in implementation. This silently breaks plural selection, locale-aware formatting, and all `Intl.NumberFormat`/`Intl.DateTimeFormat` option support. NEVER override built-in function names unless you intentionally want to replace their entire behavior.

---

## Anti-Pattern 7: Not Handling FluentNone in Custom Functions

```typescript
// WRONG -- assumes positional[0] is always a valid value
const bundle = new FluentBundle("en-US", {
  functions: {
    DOUBLE: (positional, named) => {
      return String(Number(positional[0]) * 2); // NaN if FluentNone
    },
  },
});
```

```typescript
// CORRECT -- check for FluentNone and handle gracefully
import { FluentNone } from "@fluent/bundle";

const bundle = new FluentBundle("en-US", {
  functions: {
    DOUBLE: (positional, named) => {
      const val = positional[0];
      if (val instanceof FluentNone || val === undefined) {
        return new FluentNone("DOUBLE()");
      }
      const num = typeof val === "number" ? val : Number(val);
      if (isNaN(num)) return new FluentNone("DOUBLE()");
      return String(num * 2);
    },
  },
});
```

**Why**: When a variable referenced in a function call is missing (e.g., `DOUBLE($missing)`), the resolver passes `FluentNone` as the argument. If your function does not handle this case, it produces `NaN`, `"undefined"`, or crashes. ALWAYS check for `FluentNone` and `undefined` in custom function implementations.

---

## Anti-Pattern 8: Using Functions for Static Content

```ftl
# WRONG -- using NUMBER() on a literal with no formatting options
count = You have { NUMBER(5) } items.

# CORRECT -- use the literal directly
count = You have 5 items.

# CORRECT -- use NUMBER() only when you need formatting options or variable formatting
count = You have { NUMBER($count, minimumFractionDigits: 0) } items.
```

**Why**: Calling `NUMBER()` or `DATETIME()` with a literal and no formatting options adds unnecessary function call overhead. Fluent already handles literal numbers in patterns. Use functions only when you need explicit formatting control or when formatting variables.

---

## Anti-Pattern 9: Forgetting to Repeat NUMBER() in Variant Bodies

```ftl
# WRONG -- formatting applied in selector is NOT carried to variant body
score =
    { NUMBER($score, minimumFractionDigits: 1) ->
        [0.0] You scored zero.
       *[other] You scored { $score } points.
    }
# The [other] variant shows $score WITHOUT the minimumFractionDigits: 1 formatting
# e.g., "You scored 3 points." instead of "You scored 3.0 points."

# CORRECT -- repeat NUMBER() with same options in the variant body
score =
    { NUMBER($score, minimumFractionDigits: 1) ->
        [0.0] You scored zero.
       *[other] You scored { NUMBER($score, minimumFractionDigits: 1) } points.
    }
```

**Why**: The formatting options applied to a selector expression only affect the selector matching. They do NOT automatically propagate to variant bodies. If you need the same formatting in the displayed output, you MUST call `NUMBER()` again with the same options inside the variant pattern.
