# Examples Reference -- fluent-syntax-functions

## NUMBER() Formatting Examples

### Basic Number Formatting

FTL:
```ftl
# Display with 2 decimal places (e.g., monetary display without currency symbol)
balance = Balance: { NUMBER($amount, minimumFractionDigits: 2, maximumFractionDigits: 2) }

# Display without grouping separators
raw-count = Raw: { NUMBER($count, useGrouping: 0) }

# Display with leading zeros
product-id = ID: { NUMBER($id, minimumIntegerDigits: 6) }
```

TypeScript:
```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`
balance = Balance: { NUMBER($amount, minimumFractionDigits: 2, maximumFractionDigits: 2) }
raw-count = Raw: { NUMBER($count, useGrouping: 0) }
product-id = ID: { NUMBER($id, minimumIntegerDigits: 6) }
`));

const errors: Error[] = [];

// Balance formatting
const balMsg = bundle.getMessage("balance");
if (balMsg?.value) {
  bundle.formatPattern(balMsg.value, { amount: 1234.5 }, errors);
  // → "Balance: 1,234.50"
}

// Without grouping
const rawMsg = bundle.getMessage("raw-count");
if (rawMsg?.value) {
  bundle.formatPattern(rawMsg.value, { count: 9876543 }, errors);
  // → "Raw: 9876543"
}

// Leading zeros
const idMsg = bundle.getMessage("product-id");
if (idMsg?.value) {
  bundle.formatPattern(idMsg.value, { id: 42 }, errors);
  // → "ID: 000,042"
}
```

### NUMBER() with Significant Digits

FTL:
```ftl
measurement = Result: { NUMBER($value, minimumSignificantDigits: 3, maximumSignificantDigits: 5) }
```

TypeScript:
```typescript
const msg = bundle.getMessage("measurement");
if (msg?.value) {
  bundle.formatPattern(msg.value, { value: 0.00456 }, errors);
  // → "Result: 0.00456" (3-5 significant digits preserved)

  bundle.formatPattern(msg.value, { value: 123456.789 }, errors);
  // → "Result: 123,460" (rounded to 5 significant digits)
}
```

### NUMBER() in Selectors -- Cardinal Plurals

FTL:
```ftl
items-in-cart =
    { $count ->
        [0] Your cart is empty.
        [one] You have one item in your cart.
       *[other] You have { $count } items in your cart.
    }
```

TypeScript:
```typescript
const msg = bundle.getMessage("items-in-cart");
if (msg?.value) {
  bundle.formatPattern(msg.value, { count: 0 }, errors);
  // → "Your cart is empty."

  bundle.formatPattern(msg.value, { count: 1 }, errors);
  // → "You have one item in your cart."

  bundle.formatPattern(msg.value, { count: 42 }, errors);
  // → "You have 42 items in your cart."
}
```

### NUMBER() in Selectors -- Ordinal Plurals

FTL:
```ftl
place-finish = { NUMBER($position, type: "ordinal") ->
    [1] Gold medal -- you finished first!
    [2] Silver medal -- you finished second!
    [3] Bronze medal -- you finished third!
    [one] You finished {$position}st
    [two] You finished {$position}nd
    [few] You finished {$position}rd
   *[other] You finished {$position}th
}
```

TypeScript:
```typescript
const msg = bundle.getMessage("place-finish");
if (msg?.value) {
  bundle.formatPattern(msg.value, { position: 1 }, errors);
  // → "Gold medal -- you finished first!" (exact match [1])

  bundle.formatPattern(msg.value, { position: 21 }, errors);
  // → "You finished 21st" (CLDR ordinal category [one])

  bundle.formatPattern(msg.value, { position: 22 }, errors);
  // → "You finished 22nd" (CLDR ordinal category [two])

  bundle.formatPattern(msg.value, { position: 33 }, errors);
  // → "You finished 33rd" (CLDR ordinal category [few])

  bundle.formatPattern(msg.value, { position: 45 }, errors);
  // → "You finished 45th" (CLDR ordinal category [other])
}
```

---

## DATETIME() Formatting Examples

### Date with Style Shortcuts

FTL:
```ftl
event-full = { DATETIME($date, dateStyle: "full") }
event-long = { DATETIME($date, dateStyle: "long") }
event-medium = { DATETIME($date, dateStyle: "medium") }
event-short = { DATETIME($date, dateStyle: "short") }
```

TypeScript:
```typescript
const date = new Date("2026-03-19T14:30:00");

const fullMsg = bundle.getMessage("event-full");
if (fullMsg?.value) {
  bundle.formatPattern(fullMsg.value, { date }, errors);
  // → "Thursday, March 19, 2026" (en-US)
}

const shortMsg = bundle.getMessage("event-short");
if (shortMsg?.value) {
  bundle.formatPattern(shortMsg.value, { date }, errors);
  // → "3/19/26" (en-US)
}
```

### Date with Individual Components

FTL:
```ftl
# Month and day only
birthday = Born on { DATETIME($date, month: "long", day: "numeric") }

# Full date-time with components
log-entry = { DATETIME($timestamp, year: "numeric", month: "2-digit", day: "2-digit", hour: "2-digit", minute: "2-digit", second: "2-digit") }

# Weekday display
schedule = { DATETIME($date, weekday: "long") }, { DATETIME($date, month: "short", day: "numeric") }
```

TypeScript:
```typescript
const date = new Date("2026-03-19T14:30:45");

const bdayMsg = bundle.getMessage("birthday");
if (bdayMsg?.value) {
  bundle.formatPattern(bdayMsg.value, { date }, errors);
  // → "Born on March 19"
}

const logMsg = bundle.getMessage("log-entry");
if (logMsg?.value) {
  bundle.formatPattern(logMsg.value, { timestamp: date }, errors);
  // → "03/19/2026, 02:30:45 PM" (en-US, varies by locale)
}
```

### Time-Only Formatting

FTL:
```ftl
meeting-time = Meeting at { DATETIME($time, hour: "numeric", minute: "2-digit") }
precise-time = Logged at { DATETIME($time, hour: "2-digit", minute: "2-digit", second: "2-digit", fractionalSecondDigits: 3) }
```

---

## Custom Function Registration Examples

### Text Transformation Function

TypeScript:
```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    UPCASE: (positional, named) => {
      const val = positional[0];
      return typeof val === "string"
        ? val.toUpperCase()
        : String(val).toUpperCase();
    },
    LOWERCASE: (positional, named) => {
      const val = positional[0];
      return typeof val === "string"
        ? val.toLowerCase()
        : String(val).toLowerCase();
    },
  },
});
```

FTL:
```ftl
warning = { UPCASE("attention") }: System maintenance at midnight.
username-display = Logged in as { LOWERCASE($name) }
```

### Conditional Logic Function

TypeScript:
```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    PLATFORM: (positional, named) => {
      // Return platform string for selector matching in FTL
      if (typeof navigator !== "undefined") {
        if (navigator.platform.startsWith("Win")) return "windows";
        if (navigator.platform.startsWith("Mac")) return "macos";
        return "linux";
      }
      return "unknown";
    },
  },
});
```

FTL:
```ftl
shortcut-save = { PLATFORM() ->
    [macos] Press Cmd+S to save
    [windows] Press Ctrl+S to save
   *[other] Press Ctrl+S to save
}
```

### Number Formatting Function (Currency)

TypeScript:
```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    CURRENCY: (positional, named) => {
      const value = positional[0];
      const num = typeof value === "number"
        ? value
        : value instanceof FluentNumber
          ? value.valueOf()
          : Number(value);

      const currency = named["code"];
      const currencyCode = typeof currency === "string"
        ? currency
        : "USD";

      return new Intl.NumberFormat("en-US", {
        style: "currency",
        currency: currencyCode,
      }).format(num);
    },
  },
});
```

FTL:
```ftl
total-usd = Total: { CURRENCY($amount, code: "USD") }
total-eur = Total: { CURRENCY($amount, code: "EUR") }
```

### Function with Multiple Positional Arguments

TypeScript:
```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    CONCAT: (positional, named) => {
      const separator = named["sep"];
      const sep = typeof separator === "string" ? separator : ", ";
      return positional.map(v => String(v)).join(sep);
    },
  },
});
```

FTL:
```ftl
full-name = Name: { CONCAT($first, $last, sep: " ") }
```

---

## Bidirectional Text Examples

### Default Behavior (useIsolating: true)

TypeScript:
```typescript
const bundle = new FluentBundle("ar", { useIsolating: true }); // default
bundle.addResource(new FluentResource(`
welcome = مرحبًا { $name }!
`));

const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "John" });
  // Internal: "مرحبًا \u2068John\u2069!"
  // The LTR name "John" is isolated from the surrounding RTL text
  // preventing it from disrupting the Arabic text flow
}
```

### Testing Without Isolation Marks

TypeScript:
```typescript
// For unit tests -- disable to simplify assertions
const bundle = new FluentBundle("en-US", { useIsolating: false });
bundle.addResource(new FluentResource(`greeting = Hello, { $name }!`));

const msg = bundle.getMessage("greeting");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "World" });
  // Exact string: "Hello, World!" (no Unicode marks)
  expect(text).toBe("Hello, World!"); // Clean assertion
}
```

### Mixed Directionality Scenario

FTL (Arabic locale file):
```ftl
file-info = الملف { $filename } بحجم { NUMBER($size) } بايت
```

TypeScript:
```typescript
// With useIsolating: true (default) -- CORRECT
const bundle = new FluentBundle("ar");
bundle.addResource(new FluentResource(`
file-info = الملف { $filename } بحجم { NUMBER($size) } بايت
`));

const msg = bundle.getMessage("file-info");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, {
    filename: "report.pdf",
    size: 1024,
  });
  // LTR filename and number are properly isolated within RTL text
  // Without isolation, "report.pdf" could cause "بحجم" to appear
  // on the wrong side of the number
}
```
