# Examples: Full FTL + TypeScript Flow

## Example 1: Basic Setup and Formatting

### FTL file (`en-US/messages.ftl`)

```ftl
### Application messages

hello = Hello, world!
welcome = Welcome, { $name }!
unread-emails = You have { $count } unread emails.
```

### TypeScript

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

// Step 1: Create a bundle for a locale
const bundle = new FluentBundle("en-US");

// Step 2: Parse FTL and add to bundle
const ftl = `
hello = Hello, world!
welcome = Welcome, { $name }!
unread-emails = You have { $count } unread emails.
`;
const resource = new FluentResource(ftl);
const errors = bundle.addResource(resource);
if (errors.length > 0) {
  console.error("Parse errors:", errors);
}

// Step 3: Look up and format messages
const helloMsg = bundle.getMessage("hello");
if (helloMsg?.value) {
  const text = bundle.formatPattern(helloMsg.value);
  // -> "Hello, world!"
}

const welcomeMsg = bundle.getMessage("welcome");
if (welcomeMsg?.value) {
  const text = bundle.formatPattern(welcomeMsg.value, { name: "Anna" });
  // -> "Welcome, Anna!"
}
```

---

## Example 2: Terms and Brand Names

### FTL file

```ftl
-brand-name = Firefox
-brand-short = Fx

about = About { -brand-name }.
update-available = A new version of { -brand-name } is available.
shortcut = Open { -brand-short } Settings
```

### TypeScript

```typescript
const resource = new FluentResource(`
-brand-name = Firefox
-brand-short = Fx
about = About { -brand-name }.
update-available = A new version of { -brand-name } is available.
`);
bundle.addResource(resource);

const msg = bundle.getMessage("about");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value);
  // -> "About Firefox."
}

// Terms are NOT accessible directly:
const term = bundle.getMessage("-brand-name"); // undefined
```

---

## Example 3: Selectors and Plurals

### FTL file

```ftl
emails =
    { $count ->
        [one] You have one unread email.
       *[other] You have { $count } unread emails.
    }

photos =
    { $userName } added { $photoCount ->
        [one] a new photo
       *[other] { $photoCount } new photos
    } to { $userGender ->
        [male] his stream
        [female] her stream
       *[other] their stream
    }.
```

### TypeScript

```typescript
const resource = new FluentResource(`
emails =
    { $count ->
        [one] You have one unread email.
       *[other] You have { $count } unread emails.
    }
`);
bundle.addResource(resource);

const msg = bundle.getMessage("emails");
if (msg?.value) {
  bundle.formatPattern(msg.value, { count: 1 });
  // -> "You have one unread email."

  bundle.formatPattern(msg.value, { count: 5 });
  // -> "You have 5 unread emails."
}
```

---

## Example 4: Attributes

### FTL file

```ftl
login-input = Predefined value
    .placeholder = email@example.com
    .aria-label = Login input value
    .title = Type your login email

# Value-less message (only attributes)
submit-button =
    .aria-label = Submit form
    .title = Click to submit
```

### TypeScript

```typescript
const resource = new FluentResource(`
login-input = Predefined value
    .placeholder = email@example.com
    .aria-label = Login input value
    .title = Type your login email
`);
bundle.addResource(resource);

const msg = bundle.getMessage("login-input");
if (msg) {
  // Format the main value
  if (msg.value) {
    bundle.formatPattern(msg.value);
    // -> "Predefined value"
  }

  // Format attributes
  const placeholder = bundle.formatPattern(msg.attributes["placeholder"]);
  // -> "email@example.com"

  const ariaLabel = bundle.formatPattern(msg.attributes["aria-label"]);
  // -> "Login input value"
}
```

---

## Example 5: Custom Functions

### FTL file

```ftl
greeting = Good { GREETING_TIME() }!
shouted = { SHOUT($message) }
```

### TypeScript

```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    GREETING_TIME: () => {
      const hour = new Date().getHours();
      if (hour < 12) return "morning";
      if (hour < 18) return "afternoon";
      return "evening";
    },
    SHOUT: (positional) => {
      const val = positional[0];
      return typeof val === "string"
        ? val.toUpperCase()
        : String(val).toUpperCase();
    },
  },
});

const resource = new FluentResource(`
greeting = Good { GREETING_TIME() }!
shouted = { SHOUT($message) }
`);
bundle.addResource(resource);

const msg = bundle.getMessage("greeting");
if (msg?.value) {
  bundle.formatPattern(msg.value);
  // -> "Good morning!" (if before noon)
}

const shoutMsg = bundle.getMessage("shouted");
if (shoutMsg?.value) {
  bundle.formatPattern(shoutMsg.value, { message: "hello world" });
  // -> "HELLO WORLD"
}
```

---

## Example 6: Error Handling in Production

### TypeScript

```typescript
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`
welcome = Welcome, { $name }!
`));

// ALWAYS use the errors array in production
const errors: Error[] = [];
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  // Missing $name variable -- collected as error, not thrown
  const text = bundle.formatPattern(msg.value, {}, errors);
  // text -> "Welcome, {$name}!" (fallback with placeholder)
  // errors -> [ReferenceError: Unknown variable: $name]
}

if (errors.length > 0) {
  errors.forEach((e) => console.warn("Fluent:", e.message));
}
```

---

## Example 7: Multiple Locales with Fallback

### TypeScript

```typescript
function createBundleForLocale(
  locale: string,
  ftl: string
): FluentBundle {
  const bundle = new FluentBundle(locale);
  const errors = bundle.addResource(new FluentResource(ftl));
  if (errors.length > 0) {
    console.error(`Parse errors for ${locale}:`, errors);
  }
  return bundle;
}

const bundles = [
  createBundleForLocale("nl", `welcome = Welkom, { $name }!`),
  createBundleForLocale("en-US", `welcome = Welcome, { $name }!`),
];

// Try each bundle in order until a message is found
function formatMessage(id: string, args: Record<string, string | number>): string {
  for (const bundle of bundles) {
    const msg = bundle.getMessage(id);
    if (msg?.value) {
      const errors: Error[] = [];
      return bundle.formatPattern(msg.value, args, errors);
    }
  }
  return id; // fallback to message ID
}

formatMessage("welcome", { name: "Anna" });
// -> "Welkom, Anna!" (Dutch bundle matches first)
```

---

## Example 8: Number and Date Formatting

### FTL file

```ftl
price = Total: { NUMBER($amount, minimumFractionDigits: 2) }
last-login = Last login: { DATETIME($date, dateStyle: "long") }
percent = { NUMBER($ratio, style: "percent") }
```

### TypeScript

```typescript
const resource = new FluentResource(`
price = Total: { NUMBER($amount, minimumFractionDigits: 2) }
last-login = Last login: { DATETIME($date, dateStyle: "long") }
`);
bundle.addResource(resource);

const priceMsg = bundle.getMessage("price");
if (priceMsg?.value) {
  bundle.formatPattern(priceMsg.value, { amount: 42 });
  // -> "Total: 42.00" (en-US locale)
}

const loginMsg = bundle.getMessage("last-login");
if (loginMsg?.value) {
  bundle.formatPattern(loginMsg.value, { date: new Date("2026-03-19") });
  // -> "Last login: March 19, 2026" (en-US locale)
}
```
