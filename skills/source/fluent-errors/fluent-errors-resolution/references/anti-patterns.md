# Resolution Error Anti-Patterns

## AP-01: Omitting the Errors Array in Production

```typescript
// WRONG -- throws on first resolution error, crashing the application
const text = bundle.formatPattern(pattern, args);

// CORRECT -- collect errors gracefully
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
if (errors.length > 0) {
  errors.forEach(e => reportToMonitoring(e));
}
```

**Why this matters:** Without the errors array, a single missing variable in any translation brings down the entire application. In production, partial output with fallback placeholders is ALWAYS better than a crash.

---

## AP-02: Not Checking getMessage Return Value

```typescript
// WRONG -- getMessage can return undefined
const msg = bundle.getMessage("welcome")!;
bundle.formatPattern(msg.value!, args, errors);

// CORRECT -- always guard
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  bundle.formatPattern(msg.value, args, errors);
}
```

**Why this matters:** The non-null assertion (`!`) hides two failure points: the message might not exist, and the value might be `null` (attributes-only message). Both produce runtime crashes.

---

## AP-03: Assuming getMessage Value Is Always Present

```typescript
// WRONG -- messages with only attributes have value: null
const msg = bundle.getMessage("login-input")!;
const text = bundle.formatPattern(msg.value!, args); // crashes if value is null
```

FTL messages can have ONLY attributes:
```ftl
login-input =
    .placeholder = Email address
    .aria-label = Login input
```

```typescript
// CORRECT -- check value, use attributes when needed
const msg = bundle.getMessage("login-input");
if (msg) {
  if (msg.value) {
    // Message has a main value
    bundle.formatPattern(msg.value, args, errors);
  }
  if (msg.attributes["placeholder"]) {
    // Access specific attributes
    bundle.formatPattern(msg.attributes["placeholder"], args, errors);
  }
}
```

---

## AP-04: Ignoring the Errors Array After Formatting

```typescript
// WRONG -- collecting errors but never checking them
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
return text; // errors silently discarded

// CORRECT -- always process collected errors
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
if (errors.length > 0) {
  // In development: log to console
  errors.forEach(e => console.warn("[Fluent]", e.message));
  // In production: report to monitoring
  errors.forEach(e => Sentry.captureException(e));
}
return text;
```

**Why this matters:** Without checking the errors array, missing translations silently degrade to fallback placeholders like `{$name}` in the UI. Users see broken strings with no alert to developers.

---

## AP-05: Trying to Access Terms via getMessage

```typescript
// WRONG -- terms are never returned by getMessage or hasMessage
const term = bundle.getMessage("-brand-name"); // ALWAYS returns undefined
const exists = bundle.hasMessage("-brand-name"); // ALWAYS returns false

// Terms are internal to the resolver -- they can only be referenced in FTL:
// welcome = Welcome to { -brand-name }!
// There is NO public API to format a term directly.
```

**Why this matters:** Developers often try to access terms for debugging or display. This is architecturally impossible. Terms are resolved only when referenced within message patterns.

---

## AP-06: Passing Unsupported Types as Variables

```typescript
// WRONG -- objects, arrays, booleans, null are not valid FluentVariable
bundle.formatPattern(pattern, {
  user: { name: "Anna", age: 30 },  // TypeError: object
  items: [1, 2, 3],                  // TypeError: array
  active: true,                       // TypeError: boolean
  data: null,                         // TypeError: null
});

// CORRECT -- only string, number, Date, FluentType, or Temporal objects
bundle.formatPattern(pattern, {
  userName: "Anna",                             // string
  age: 30,                                      // number
  loginDate: new Date(),                         // Date
  price: new FluentNumber(9.99, { minimumFractionDigits: 2 }), // FluentType
});
```

**Why this matters:** Invalid types produce TypeError resolution errors. The affected placeables show fallback values like `{$user}` instead of meaningful content.

---

## AP-07: Reusing the Same Errors Array Across Multiple Calls

```typescript
// MISLEADING -- errors accumulate across calls
const errors: Error[] = [];
const text1 = bundle.formatPattern(pattern1, args1, errors);
const text2 = bundle.formatPattern(pattern2, args2, errors);
// errors now contains errors from BOTH calls -- you cannot tell which is which

// CORRECT -- fresh array per call when you need per-message diagnostics
const errors1: Error[] = [];
const text1 = bundle.formatPattern(pattern1, args1, errors1);

const errors2: Error[] = [];
const text2 = bundle.formatPattern(pattern2, args2, errors2);
```

**Why this matters:** When errors accumulate, you cannot determine which message caused which error. This makes debugging impossible in batch formatting scenarios.

---

## AP-08: Confusing Parse Errors with Resolution Errors

```typescript
// WRONG -- treating addResource errors as resolution errors
const resource = new FluentResource(ftlContent);
const errors = bundle.addResource(resource);
if (errors.length > 0) {
  // These are PARSE errors (syntax), NOT resolution errors
  // Other valid messages in the resource were still added successfully
}

// ALSO WRONG -- expecting formatPattern to report parse errors
const fmtErrors: Error[] = [];
bundle.formatPattern(msg.value!, args, fmtErrors);
// fmtErrors only contains RESOLUTION errors (missing refs, wrong types)
// It does NOT contain syntax errors from parsing
```

**Why this matters:** Parse errors and resolution errors are completely independent error surfaces. A message can parse successfully but fail at resolution. Conversely, parse errors in one message do not affect resolution of other messages.

---

## AP-09: Not Providing a Default Variant in Select Expressions

```ftl
# WRONG -- no default variant
items = { $count ->
    [one] One item
    [few] A few items
}
# If $count matches neither "one" nor "few", output is {???}

# CORRECT -- always include *[other]
items = { $count ->
    [one] One item
    [few] A few items
   *[other] { $count } items
}
```

**Why this matters:** Without a default variant, any value that does not match an explicit key produces `{???}` and a RangeError. The `*[other]` variant is required by the Fluent specification for every select expression.

---

## AP-10: Silencing ReactLocalization reportError

```typescript
// WRONG -- swallowing all errors
const l10n = new ReactLocalization(bundles, null, () => {});

// CORRECT -- route errors to monitoring
const l10n = new ReactLocalization(bundles, null, (error) => {
  console.warn("[Fluent]", error.message);
  monitoringService.captureError(error);
});
```

**Why this matters:** The `reportError` callback is your only signal that translations are failing in the React layer. Silencing it means broken translations go undetected in production.

---

## AP-11: Using hasMessage as Sufficient Validation

```typescript
// WRONG -- hasMessage only checks existence, not completeness
if (bundle.hasMessage("welcome")) {
  const msg = bundle.getMessage("welcome")!;
  const text = bundle.formatPattern(msg.value!, args); // may still crash
}

// CORRECT -- full validation chain
if (bundle.hasMessage("welcome")) {
  const msg = bundle.getMessage("welcome")!;
  if (msg.value) {
    const errors: Error[] = [];
    const text = bundle.formatPattern(msg.value, args, errors);
    // Handle errors
  }
}
```

**Why this matters:** `hasMessage` confirms the ID exists but says nothing about whether the message has a value (vs. only attributes) or whether resolution will succeed. ALWAYS check `msg.value` and provide an errors array.
