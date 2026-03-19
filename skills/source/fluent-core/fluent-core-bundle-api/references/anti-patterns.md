# FluentBundle API — Anti-Patterns

## Anti-Pattern 1: Passing FTL Strings Directly to addResource

```typescript
// WRONG — addResource expects a FluentResource, not a string
bundle.addResource(`hello = Hello`); // TypeError at runtime

// CORRECT — ALWAYS wrap in FluentResource
bundle.addResource(new FluentResource(`hello = Hello`));
```

**Why this happens**: Developers migrating from the pre-0.14.0 API where `addMessages()` accepted raw strings.

---

## Anti-Pattern 2: Ignoring the Two-Step getMessage + formatPattern Flow

```typescript
// WRONG — there is no bundle.format() method (removed in v0.14.0)
bundle.format("welcome", { name: "Anna" }); // TypeError: not a function

// CORRECT — two-step: get message, then format its pattern
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  bundle.formatPattern(msg.value, { name: "Anna" });
}
```

**Why this matters**: The v0.14.0 API overhaul split formatting into two steps to allow null-checking and attribute access. There is no shortcut.

---

## Anti-Pattern 3: Not Checking for Null Value

```typescript
// DANGEROUS — messages can have null values (attributes-only messages)
const msg = bundle.getMessage("login-input")!;
bundle.formatPattern(msg.value!, args); // Runtime error if value is null

// CORRECT — ALWAYS check value before formatting
const msg = bundle.getMessage("login-input");
if (msg?.value) {
  bundle.formatPattern(msg.value, args);
}
```

**Why this matters**: Messages with only attributes (e.g., `login-input = .placeholder = Email`) have `value: null`. Using the non-null assertion operator `!` hides this and causes runtime crashes.

---

## Anti-Pattern 4: Omitting the Errors Array in Production

```typescript
// DANGEROUS — throws on first resolution error, crashing the app
const text = bundle.formatPattern(pattern, args);

// CORRECT for production — collect errors, show best-effort text
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
if (errors.length > 0) {
  console.warn("Formatting issues:", errors);
}
```

**Why this matters**: Without the errors array, `formatPattern` throws a `ReferenceError`, `TypeError`, or `RangeError` on the first resolution problem. In production, a missing variable should show `{$name}` fallback text, not crash the entire page.

---

## Anti-Pattern 5: Trying to Access Terms via getMessage

```typescript
// WRONG — terms (prefixed with -) are NOT accessible via getMessage
const term = bundle.getMessage("-brand-name"); // returns undefined

// Terms are resolved automatically when referenced in messages:
// FTL: welcome = Welcome to {-brand-name}!
// There is NO public API to directly format a term.
```

**Why this matters**: Terms are internal-only references in Fluent. They are resolved during pattern formatting but cannot be retrieved or formatted independently. This is by design -- terms provide reusable fragments without exposing them as standalone messages.

---

## Anti-Pattern 6: Creating FluentResource per Message

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

**Why this matters**: Each `FluentResource` constructor call initializes a new parser instance. Batching messages into a single FTL string and parsing once is significantly more efficient, especially for large translation files.

---

## Anti-Pattern 7: Using @fluent/syntax for Runtime Formatting

```typescript
// WRONG — @fluent/syntax is a tooling library, too heavy for runtime
import { FluentParser } from "@fluent/syntax";
const ast = new FluentParser().parse(ftlSource);
// ...manually walking the AST to format messages

// CORRECT — use @fluent/bundle for runtime formatting
import { FluentBundle, FluentResource } from "@fluent/bundle";
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftlSource));
```

**Why this matters**: `@fluent/bundle` includes its own optimized runtime parser that is smaller and faster than `@fluent/syntax`. The `@fluent/syntax` package is designed for tooling: linting, programmatic FTL generation, and editor integrations. Using it for runtime formatting adds unnecessary bundle size and complexity.

---

## Anti-Pattern 8: Passing Unsupported Types as Variables

```typescript
// WRONG — objects, arrays, and booleans are NOT valid FluentVariable types
bundle.formatPattern(pattern, { user: { name: "Anna" } }); // TypeError
bundle.formatPattern(pattern, { items: [1, 2, 3] });       // TypeError
bundle.formatPattern(pattern, { active: true });            // TypeError

// CORRECT — only string, number, Date, FluentType, or TemporalObject
bundle.formatPattern(pattern, {
  name: "Anna",           // string
  count: 5,               // number
  date: new Date(),       // Date
  amount: new FluentNumber(42.5, { minimumFractionDigits: 2 }), // FluentType
});
```

**Why this matters**: The `FluentVariable` type is strictly defined as `string | number | Date | FluentType | TemporalObject`. Passing other JavaScript types causes a `TypeError` during resolution. ALWAYS extract primitive values from complex objects before passing them as variables.

---

## Anti-Pattern 9: Not Handling addResource Errors

```typescript
// WRONG — silently ignoring parse errors
bundle.addResource(new FluentResource(userProvidedFtl));

// CORRECT — ALWAYS check and log errors
const errors = bundle.addResource(new FluentResource(userProvidedFtl));
if (errors.length > 0) {
  errors.forEach(e => console.error("FTL parse error:", e.message));
}
```

**Why this matters**: `addResource()` returns an array of errors for messages with syntax problems. While other valid messages in the same resource are still added, ignoring errors means silently missing broken translations. This is especially dangerous with user-provided or dynamically loaded FTL content.

---

## Anti-Pattern 10: Disabling useIsolating Without Understanding Bidi

```typescript
// RISKY — disabling isolation marks can break RTL/LTR mixed text
const bundle = new FluentBundle("ar", { useIsolating: false });
// Arabic text with English names will display incorrectly

// SAFE — only disable when content is unidirectional
const bundle = new FluentBundle("en-US", { useIsolating: false });

// SAFE — disable for testing when Unicode marks interfere with assertions
const testBundle = new FluentBundle("en-US", { useIsolating: false });
```

**Why this matters**: The `useIsolating` option wraps placeables in Unicode bidi isolation marks (U+2068 First Strong Isolate and U+2069 Pop Directional Isolate). This prevents inserted values from affecting the directionality of surrounding text. Disabling it for bidirectional locales (Arabic, Hebrew, etc.) causes layout corruption when LTR content (names, numbers) appears in RTL text. ONLY disable this when you are certain all content flows in one direction, or during testing.
