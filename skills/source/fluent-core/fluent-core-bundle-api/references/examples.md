# FluentBundle API — Working Examples

## Bundle Creation

### Basic Bundle

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
const resource = new FluentResource(`
hello = Hello, world!
goodbye = Goodbye, {$name}!
`);

const errors = bundle.addResource(resource);
if (errors.length > 0) {
  console.error("FTL syntax errors:", errors);
}
```

### Bundle with Multiple Locales

```typescript
// First locale is primary; others are fallbacks for Intl formatters
const bundle = new FluentBundle(["en-US", "en-GB"]);
```

### Bundle with All Options

```typescript
import { FluentBundle, FluentResource, FluentValue } from "@fluent/bundle";

const bundle = new FluentBundle("en-US", {
  useIsolating: false, // Disable bidi isolation marks
  functions: {
    UPCASE: (positional: FluentValue[], named: Record<string, FluentValue>) => {
      const val = positional[0];
      return typeof val === "string" ? val.toUpperCase() : String(val).toUpperCase();
    },
    GREETING_TIME: () => {
      const hour = new Date().getHours();
      if (hour < 12) return "morning";
      if (hour < 18) return "afternoon";
      return "evening";
    },
  },
  transform: (text) => text, // Identity transform (default)
});
```

---

## Resource Loading

### Single Resource

```typescript
const resource = new FluentResource(`
welcome = Welcome, {$name}!
items-count = You have {$count} items.
`);

const errors = bundle.addResource(resource);
// ALWAYS check errors — they indicate FTL syntax problems
if (errors.length > 0) {
  errors.forEach(e => console.error("Parse error:", e.message));
}
```

### Multiple Resources with Override Control

```typescript
const base = new FluentResource(`
hello = Hello
goodbye = Goodbye
`);

const brandOverride = new FluentResource(`
hello = Hi there
`);

bundle.addResource(base);

// Without allowOverrides: duplicate "hello" produces an error
const errs1 = bundle.addResource(brandOverride);
// errs1 contains an error about duplicate "hello"

// With allowOverrides: "hello" is replaced silently
bundle.addResource(brandOverride, { allowOverrides: true });
// "hello" now resolves to "Hi there"
```

### Loading FTL from a File (Node.js)

```typescript
import { readFile } from "fs/promises";
import { FluentBundle, FluentResource } from "@fluent/bundle";

async function loadBundle(locale: string): Promise<FluentBundle> {
  const bundle = new FluentBundle(locale);
  const ftl = await readFile(`./locales/${locale}/messages.ftl`, "utf-8");
  const errors = bundle.addResource(new FluentResource(ftl));
  if (errors.length > 0) {
    console.error(`Parse errors in ${locale}:`, errors);
  }
  return bundle;
}
```

---

## Message Formatting

### Basic Variable Substitution

```typescript
const resource = new FluentResource(`
welcome = Welcome, {$name}, to {-brand-name}!
-brand-name = Acme Corp
`);
bundle.addResource(resource);

const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const errors: Error[] = [];
  const text = bundle.formatPattern(msg.value, { name: "Anna" }, errors);
  // text === "Welcome, Anna, to Acme Corp!"
}
```

### Formatting Attributes

```typescript
const resource = new FluentResource(`
login-input =
    .placeholder = Email address
    .aria-label = Login input field
`);
bundle.addResource(resource);

const msg = bundle.getMessage("login-input");
if (msg) {
  // msg.value is null — this message only has attributes
  const placeholder = bundle.formatPattern(msg.attributes["placeholder"]);
  // placeholder === "Email address"

  const ariaLabel = bundle.formatPattern(msg.attributes["aria-label"]);
  // ariaLabel === "Login input field"
}
```

### Formatting with Numbers

```typescript
const resource = new FluentResource(`
items-count = You have {$count} items.
price = Total: {NUMBER($amount, minimumFractionDigits: 2)}
`);
bundle.addResource(resource);

const msg = bundle.getMessage("items-count");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { count: 5 });
  // text === "You have 5 items."
}

const priceMsg = bundle.getMessage("price");
if (priceMsg?.value) {
  const text = bundle.formatPattern(priceMsg.value, { amount: 42.5 });
  // text === "Total: 42.50" (en-US locale)
}
```

### Formatting with Dates

```typescript
const resource = new FluentResource(`
last-login = Last login: {DATETIME($date, dateStyle: "long")}
event-time = Event at {DATETIME($date, hour: "numeric", minute: "numeric")}
`);
bundle.addResource(resource);

const msg = bundle.getMessage("last-login");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { date: new Date("2026-03-19") });
  // text === "Last login: March 19, 2026" (en-US locale)
}
```

### Formatting with FluentNumber for Pre-configured Options

```typescript
import { FluentBundle, FluentResource, FluentNumber } from "@fluent/bundle";

const resource = new FluentResource(`
balance = Account balance: {$amount}
`);
bundle.addResource(resource);

const msg = bundle.getMessage("balance");
if (msg?.value) {
  const amount = new FluentNumber(1234.5, {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  });
  const text = bundle.formatPattern(msg.value, { amount });
  // text === "Account balance: 1,234.50" (en-US locale)
}
```

---

## Error Collection

### Production Pattern: Collect and Log

```typescript
function formatMessage(
  bundle: FluentBundle,
  id: string,
  args?: Record<string, FluentVariable>
): string {
  const msg = bundle.getMessage(id);
  if (!msg) {
    console.warn(`Missing message: ${id}`);
    return id; // Return the message ID as fallback
  }

  if (!msg.value) {
    console.warn(`Message "${id}" has no value (attributes-only)`);
    return id;
  }

  const errors: Error[] = [];
  const text = bundle.formatPattern(msg.value, args ?? null, errors);

  if (errors.length > 0) {
    errors.forEach(e => console.warn(`Fluent error in "${id}":`, e.message));
  }

  return text;
}
```

### Development Pattern: Throw on Error

```typescript
// Useful during development to catch missing variables early
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  try {
    // No errors array — throws on first resolution error
    const text = bundle.formatPattern(msg.value, { name: "Anna" });
  } catch (e) {
    console.error("Resolution error:", e);
  }
}
```

### Checking addResource Errors

```typescript
const ftlSource = await fetchFtlFromServer();
const resource = new FluentResource(ftlSource);
const parseErrors = bundle.addResource(resource);

if (parseErrors.length > 0) {
  // Some messages had syntax errors but others were added successfully
  parseErrors.forEach(e => {
    console.error("FTL parse error:", e.message);
  });
}
```

---

## Custom Functions

### Registering and Using Custom Functions

```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    // Simple string transformation
    SHOUT: (positional) => {
      const val = positional[0];
      return typeof val === "string" ? val.toUpperCase() : String(val).toUpperCase();
    },

    // Function with named arguments
    CURRENCY: (positional, named) => {
      const amount = positional[0];
      const currency = named["currency"];
      const numVal = typeof amount === "number" ? amount
        : amount instanceof FluentNumber ? amount.value
        : Number(amount);
      return new Intl.NumberFormat("en-US", {
        style: "currency",
        currency: typeof currency === "string" ? currency : "USD",
      }).format(numVal);
    },

    // Time-based function (no arguments needed)
    GREETING_TIME: () => {
      const hour = new Date().getHours();
      if (hour < 12) return "morning";
      if (hour < 18) return "afternoon";
      return "evening";
    },
  },
});

const resource = new FluentResource(`
shout = {SHOUT($name)}!
price = {CURRENCY($amount, currency: "EUR")}
greeting = Good {GREETING_TIME()}!
`);
bundle.addResource(resource);
```

### Using Built-in NUMBER Options in FTL

```ftl
# NUMBER options are passed through to Intl.NumberFormat
percentage = {NUMBER($value, style: "percent")}
precise = {NUMBER($value, minimumFractionDigits: 4)}
grouped = {NUMBER($value, useGrouping: "always")}
```

### Using Built-in DATETIME Options in FTL

```ftl
# DATETIME options are passed through to Intl.DateTimeFormat
short-date = {DATETIME($date, dateStyle: "short")}
full-date = {DATETIME($date, dateStyle: "full")}
time-only = {DATETIME($date, hour: "numeric", minute: "numeric")}
weekday = {DATETIME($date, weekday: "long")}
```

---

## Complete Integration Example

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

// 1. Create bundle for locale
const bundle = new FluentBundle("en-US", {
  useIsolating: false, // Disable for testing clarity
  functions: {
    PLATFORM: () => process.platform === "win32" ? "Windows" : "Other",
  },
});

// 2. Load FTL resource
const ftl = `
app-title = My Application
welcome = Welcome back, {$username}!
items =
    { $count ->
        [one] You have {$count} item
       *[other] You have {$count} items
    }
platform-info = Running on {PLATFORM()}
save-button =
    .label = Save
    .accesskey = S
    .title = Save your changes
`;

const parseErrors = bundle.addResource(new FluentResource(ftl));
if (parseErrors.length > 0) {
  console.error("Parse errors:", parseErrors);
}

// 3. Format messages with full error handling
function t(id: string, args?: Record<string, any>): string {
  const msg = bundle.getMessage(id);
  if (!msg) return id;
  if (!msg.value) return id;

  const errors: Error[] = [];
  const result = bundle.formatPattern(msg.value, args ?? null, errors);
  if (errors.length > 0) {
    console.warn(`[i18n] Errors formatting "${id}":`, errors);
  }
  return result;
}

function tAttr(id: string, attr: string, args?: Record<string, any>): string {
  const msg = bundle.getMessage(id);
  if (!msg) return `${id}.${attr}`;

  const pattern = msg.attributes[attr];
  if (!pattern) return `${id}.${attr}`;

  const errors: Error[] = [];
  const result = bundle.formatPattern(pattern, args ?? null, errors);
  if (errors.length > 0) {
    console.warn(`[i18n] Errors formatting "${id}.${attr}":`, errors);
  }
  return result;
}

// Usage
t("welcome", { username: "Anna" });     // "Welcome back, Anna!"
t("items", { count: 1 });               // "You have 1 item"
t("items", { count: 5 });               // "You have 5 items"
t("platform-info");                      // "Running on Windows"
tAttr("save-button", "label");           // "Save"
tAttr("save-button", "title");           // "Save your changes"
```
