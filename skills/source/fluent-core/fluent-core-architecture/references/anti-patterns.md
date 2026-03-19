# Anti-Patterns

## Anti-Pattern 1: Confusing Fluent with ICU MessageFormat

Fluent and ICU MessageFormat are fundamentally different systems. NEVER apply ICU patterns to Fluent.

| Mistake | ICU Thinking | Fluent Reality |
|---------|-------------|----------------|
| Using `{count, plural, ...}` syntax | ICU plural syntax | Fluent uses `{ $count -> [one] ... *[other] ... }` |
| Expecting symmetric translations | All locales mirror source structure | Each locale develops independently |
| Modifying source strings for target grammar | Developer adds `select` for Polish cases | Polish translator adds selectors independently |
| Deeply nesting select expressions | Common in ICU | Fluent favors flat variants |

### Wrong Mental Model

```
ICU thinking: "I need to add a gender parameter to my source string
so the German translator can use it."

Fluent thinking: "The German translator adds a selector on a term
attribute. The source string and code stay unchanged."
```

---

## Anti-Pattern 2: Passing Raw FTL Strings to addResource

```typescript
// WRONG -- addResource expects a FluentResource, not a string
bundle.addResource(`hello = Hello`); // TypeError

// CORRECT
bundle.addResource(new FluentResource(`hello = Hello`));
```

This mistake is common among developers migrating from the pre-0.14.0 API where `addMessages()` accepted raw strings.

---

## Anti-Pattern 3: Calling a Non-Existent format() Method

```typescript
// WRONG -- there is no bundle.format() method (removed in v0.14.0)
bundle.format("welcome", { name: "Anna" }); // TypeError

// CORRECT -- two-step: get message, then format its pattern
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  bundle.formatPattern(msg.value, { name: "Anna" });
}
```

ALWAYS use the two-step `getMessage()` + `formatPattern()` flow.

---

## Anti-Pattern 4: Not Checking for null Value

```typescript
// DANGEROUS -- messages can have null values (attributes-only messages)
const msg = bundle.getMessage("login-input")!;
bundle.formatPattern(msg.value!, args); // Runtime error if value is null

// CORRECT -- always check
const msg = bundle.getMessage("login-input");
if (msg?.value) {
  bundle.formatPattern(msg.value, args);
}
```

Messages with only attributes (no main value) have `value: null`. ALWAYS guard with a null check.

---

## Anti-Pattern 5: Omitting the Errors Array in Production

```typescript
// DANGEROUS -- throws on first resolution error, crashing the app
const text = bundle.formatPattern(pattern, args);

// CORRECT for production -- collect errors gracefully
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
if (errors.length > 0) {
  errors.forEach((e) => console.warn("Fluent:", e.message));
}
```

Without an errors array, `formatPattern()` throws a `ReferenceError`, `TypeError`, or `RangeError` on the first resolution problem. In production, ALWAYS pass an errors array so the user sees a best-effort string with `{???}` placeholders instead of a crash.

---

## Anti-Pattern 6: Trying to Access Terms via getMessage

```typescript
// WRONG -- terms (prefixed with -) are not accessible via getMessage
const term = bundle.getMessage("-brand-name"); // returns undefined

// Terms are resolved automatically when referenced in messages:
// welcome = Welcome to { -brand-name }!
// There is NO public API to directly format a term.
```

Terms are internal to the localization system. They exist for translator consistency, not for direct runtime access.

---

## Anti-Pattern 7: Creating One FluentResource Per Message

```typescript
// WRONG -- creates parser overhead per message
for (const [id, text] of Object.entries(messages)) {
  bundle.addResource(new FluentResource(`${id} = ${text}`));
}

// CORRECT -- batch all messages into one resource
const ftl = Object.entries(messages)
  .map(([id, text]) => `${id} = ${text}`)
  .join("\n");
bundle.addResource(new FluentResource(ftl));
```

Each `FluentResource` instantiation invokes the parser. ALWAYS batch messages into a single FTL string per resource.

---

## Anti-Pattern 8: Using @fluent/syntax for Runtime Formatting

```typescript
// WRONG -- @fluent/syntax is a tooling library, too heavy for runtime
import { FluentParser } from "@fluent/syntax";
const ast = new FluentParser().parse(ftlSource);
// ... manually walking the AST to format messages

// CORRECT -- use @fluent/bundle for runtime
import { FluentBundle, FluentResource } from "@fluent/bundle";
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftlSource));
```

`@fluent/bundle` includes its own optimized runtime parser. `@fluent/syntax` is for tooling: linting, programmatic FTL generation, and editor integrations.

---

## Anti-Pattern 9: Passing Unsupported Types as Variables

```typescript
// WRONG -- objects, arrays, booleans are NOT valid FluentVariable types
bundle.formatPattern(pattern, { user: { name: "Anna" } }); // TypeError
bundle.formatPattern(pattern, { active: true }); // TypeError

// CORRECT -- only string, number, Date, FluentType, or Temporal objects
bundle.formatPattern(pattern, { name: "Anna", count: 5, date: new Date() });
```

`FluentVariable` accepts: `string`, `number`, `Date`, `FluentType` subclasses, and Temporal objects (v0.19.0+). Everything else causes a `TypeError`.

---

## Anti-Pattern 10: Ignoring addResource Errors

```typescript
// WRONG -- silently ignoring parse errors
bundle.addResource(new FluentResource(userProvidedFtl));

// CORRECT -- check and log errors
const errors = bundle.addResource(new FluentResource(userProvidedFtl));
if (errors.length > 0) {
  errors.forEach((e) => console.error("FTL parse error:", e.message));
}
```

`addResource()` returns an array of errors for messages with syntax problems. These messages are silently skipped. ALWAYS check the return value to catch malformed FTL early.

---

## Anti-Pattern 11: Disabling useIsolating Without Understanding Bidi

```typescript
// RISKY -- disabling isolation marks can break RTL/LTR mixed text
const bundle = new FluentBundle("ar", { useIsolating: false });

// ONLY disable when content is unidirectional or during testing
const bundle = new FluentBundle("en-US", { useIsolating: false });
```

The `useIsolating` option wraps placeables in Unicode bidi isolation marks (U+2068/U+2069). Disabling it for RTL locales causes text display corruption when placeables contain LTR content (like numbers or brand names). NEVER disable for mixed-direction content.
