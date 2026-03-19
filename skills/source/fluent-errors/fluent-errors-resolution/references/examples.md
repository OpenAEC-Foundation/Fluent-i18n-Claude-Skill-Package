# Resolution Error Examples

Each example shows the error type, FTL that triggers it, TypeScript code demonstrating the cause, and the fix.

---

## Error 1: Unknown Variable

### Cause
```ftl
welcome = Welcome, { $userName }!
```

```typescript
const msg = bundle.getMessage("welcome")!;
const errors: Error[] = [];
// Missing the userName variable in args
const text = bundle.formatPattern(msg.value!, {}, errors);
// text = "Welcome, {$userName}!"
// errors = [ReferenceError: Unknown variable: $userName]
```

### Fix
```typescript
const text = bundle.formatPattern(msg.value!, { userName: "Anna" }, errors);
// text = "Welcome, Anna!"
// errors = []
```

---

## Error 2: Unknown Message Reference

### Cause
```ftl
greeting = { welcome-back }, friend!
# welcome-back is NOT defined in any loaded resource
```

```typescript
const msg = bundle.getMessage("greeting")!;
const errors: Error[] = [];
const text = bundle.formatPattern(msg.value!, null, errors);
// text = "{welcome-back}, friend!"
// errors = [ReferenceError: Unknown message: welcome-back]
```

### Fix
```ftl
welcome-back = Welcome back
greeting = { welcome-back }, friend!
```

---

## Error 3: Unknown Term Reference

### Cause
```ftl
about = Learn more about { -product-name }.
# -product-name is NOT defined
```

```typescript
const msg = bundle.getMessage("about")!;
const errors: Error[] = [];
const text = bundle.formatPattern(msg.value!, null, errors);
// text = "Learn more about {-product-name}."
// errors = [ReferenceError: Unknown term: -product-name]
```

### Fix
```ftl
-product-name = Fluent
about = Learn more about { -product-name }.
```

---

## Error 4: Unknown Attribute Reference

### Cause
```ftl
login-input =
    .placeholder = Email address

# Referencing a non-existent attribute
hint = Type in the { login-input.tooltip } field.
```

```typescript
const msg = bundle.getMessage("hint")!;
const errors: Error[] = [];
const text = bundle.formatPattern(msg.value!, null, errors);
// text = "Type in the {login-input.tooltip} field."
// errors = [ReferenceError: Unknown attribute: login-input.tooltip]
```

### Fix
```ftl
login-input =
    .placeholder = Email address
    .tooltip = Enter your email

hint = Type in the { login-input.tooltip } field.
```

---

## Error 5: No Value on Message (Attributes Only)

### Cause
```ftl
# This message has NO value, only attributes
search-input =
    .placeholder = Search...
    .aria-label = Search field

# Referencing a message without a value
search-prompt = Use the { search-input } to find items.
```

```typescript
// Direct formatting also fails
const msg = bundle.getMessage("search-input")!;
// msg.value is null!
if (msg.value) {
  bundle.formatPattern(msg.value, null, errors);
} else {
  // Handle: message has no value pattern
  // Use attributes instead:
  const placeholder = bundle.formatPattern(msg.attributes["placeholder"]!);
}
```

### Fix (Option A: Add a value)
```ftl
search-input = Search
    .placeholder = Search...
    .aria-label = Search field
```

### Fix (Option B: Reference the attribute directly)
```ftl
search-prompt = Use the { search-input.placeholder } to find items.
```

---

## Error 6: Unknown Function

### Cause
```ftl
greeting = { GREETING_TIME() }, user!
```

```typescript
// Bundle created WITHOUT registering GREETING_TIME
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`greeting = { GREETING_TIME() }, user!`));

const msg = bundle.getMessage("greeting")!;
const errors: Error[] = [];
const text = bundle.formatPattern(msg.value!, null, errors);
// text = "{GREETING_TIME()}, user!"
// errors = [ReferenceError: Unknown function: GREETING_TIME()]
```

### Fix
```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    GREETING_TIME: () => {
      const hour = new Date().getHours();
      if (hour < 12) return "Good morning";
      if (hour < 18) return "Good afternoon";
      return "Good evening";
    },
  },
});
```

---

## Error 7: Unsupported Variable Type

### Cause
```ftl
user-info = Name: { $user }
```

```typescript
const msg = bundle.getMessage("user-info")!;
const errors: Error[] = [];
// Objects are NOT valid FluentVariable types
const text = bundle.formatPattern(msg.value!, { user: { name: "Anna" } }, errors);
// text = "Name: {$user}"
// errors = [TypeError]
```

### Fix
```typescript
// Only string, number, Date, FluentType, or Temporal objects are valid
const text = bundle.formatPattern(msg.value!, { user: "Anna" }, errors);
// text = "Name: Anna"
```

### Valid FluentVariable Types
```typescript
// All valid argument types:
bundle.formatPattern(pattern, {
  name: "Anna",           // string
  count: 42,              // number
  date: new Date(),       // Date
  price: new FluentNumber(9.99, { minimumFractionDigits: 2 }), // FluentType
}, errors);

// INVALID types that cause TypeError:
// { user: { name: "x" } }   -- object
// { items: [1, 2, 3] }      -- array
// { active: true }           -- boolean
// { data: null }             -- null
// { fn: () => {} }           -- function
```

---

## Error 8: Function Not Callable

### Cause
```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    // Registered as a string instead of a function
    UPCASE: "not a function" as any,
  },
});
```

### Fix
```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    UPCASE: (positional: FluentValue[], named: Record<string, FluentValue>) => {
      const val = positional[0];
      return typeof val === "string" ? val.toUpperCase() : String(val).toUpperCase();
    },
  },
});
```

---

## Error 9: Cyclic Reference

### Cause
```ftl
message-a = See { message-b } for details.
message-b = Refer to { message-a } for context.
```

```typescript
const msg = bundle.getMessage("message-a")!;
const errors: Error[] = [];
const text = bundle.formatPattern(msg.value!, null, errors);
// text = "See Refer to {???} for context. for details."
// errors = [RangeError: Cyclic reference]
```

### Fix
Break the circular dependency:
```ftl
message-a = See the documentation for details.
message-b = Refer to { message-a } for context.
```

---

## Error 10: No Default Variant

### Cause
```ftl
# Missing the *[other] default variant
items = { $count ->
    [one] One item
    [few] A few items
}
```

```typescript
const msg = bundle.getMessage("items")!;
const errors: Error[] = [];
const text = bundle.formatPattern(msg.value!, { count: 5 }, errors);
// text = "{???}"
// errors = [RangeError: No default variant]
```

### Fix
```ftl
items = { $count ->
    [one] One item
    [few] A few items
   *[other] { $count } items
}
```

ALWAYS include a `*[other]` default variant in every select expression.

---

## Error 11: Excessive Placeables (FATAL)

### Cause
A pattern containing more than 100 placeables. This is extremely rare in hand-written FTL but can occur with programmatic generation:

```typescript
// Hypothetical: a generated FTL message with 101+ placeables
const ftl = `msg = ${Array.from({length: 101}, (_, i) => `{$v${i}}`).join(" ")}`;
bundle.addResource(new FluentResource(ftl));

const msg = bundle.getMessage("msg")!;
const errors: Error[] = [];
// This ALWAYS throws, even with errors array provided
try {
  bundle.formatPattern(msg.value!, args, errors);
} catch (e) {
  // RangeError: Too many placeables
}
```

### Fix
Refactor into multiple smaller messages:
```ftl
part-1 = { $v0 } { $v1 } { $v2 } ...
part-2 = { $v50 } { $v51 } { $v52 } ...
```
Compose in application code, not in a single FTL pattern.

---

## Complete Debugging Workflow

```typescript
function debugTranslation(
  bundle: FluentBundle,
  id: string,
  args?: Record<string, FluentVariable>
): string {
  // Step 1: Check message existence
  if (!bundle.hasMessage(id)) {
    console.error(`[Fluent] Message "${id}" not found in bundle.`);
    console.error(`  - Is the FTL file loaded?`);
    console.error(`  - Is the message ID spelled correctly?`);
    console.error(`  - Is it a term (prefixed with -)? Terms cannot be accessed directly.`);
    return id;
  }

  // Step 2: Retrieve message
  const msg = bundle.getMessage(id)!;

  // Step 3: Check for value
  if (!msg.value) {
    console.error(`[Fluent] Message "${id}" has no value (attributes only).`);
    console.error(`  Available attributes: ${Object.keys(msg.attributes).join(", ")}`);
    return id;
  }

  // Step 4: Format with error collection
  const errors: Error[] = [];
  const text = bundle.formatPattern(msg.value, args ?? null, errors);

  // Step 5: Report any resolution errors
  if (errors.length > 0) {
    console.error(`[Fluent] Resolution errors for "${id}":`);
    errors.forEach((e, i) => {
      console.error(`  ${i + 1}. [${e.constructor.name}] ${e.message}`);
    });
    console.error(`  Output (with fallbacks): "${text}"`);
    console.error(`  Args provided: ${JSON.stringify(args)}`);
  }

  return text;
}
```

### Usage

```typescript
// Development mode: use the debug function
const text = debugTranslation(bundle, "welcome-user", { userName: "Anna" });

// Production mode: use standard error collection
const errors: Error[] = [];
const msg = bundle.getMessage("welcome-user");
const text = msg?.value
  ? bundle.formatPattern(msg.value, { userName: "Anna" }, errors)
  : "welcome-user";
if (errors.length > 0) {
  reportToMonitoring(errors);
}
```
