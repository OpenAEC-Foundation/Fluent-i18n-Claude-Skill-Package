# fluent-errors-parsing — Anti-Patterns Reference

## AP-1: Ignoring addResource() Errors

### Wrong

```typescript
const bundle = new FluentBundle("en-US");
const resource = new FluentResource(userProvidedFtl);
bundle.addResource(resource);
// No error checking — broken messages silently disappear
```

### Why This Is Dangerous

- `addResource()` returns `Array<Error>` — one error per broken message
- Valid messages still load, so the bundle appears to work
- Broken messages simply vanish — `getMessage()` returns `undefined`
- In production, users see missing translations with no indication of why
- Debugging becomes extremely difficult without error logging

### Correct

```typescript
const bundle = new FluentBundle("en-US");
const resource = new FluentResource(userProvidedFtl);
const errors = bundle.addResource(resource);

if (errors.length > 0) {
  for (const err of errors) {
    console.error("FTL parse error:", err.message);
  }
  // In development: throw or display warnings prominently
  // In production: log to error monitoring service
}
```

---

## AP-2: Using @fluent/syntax for Runtime Formatting

### Wrong

```typescript
import { FluentParser } from "@fluent/syntax";

// Parsing FTL into full AST for runtime use
const parser = new FluentParser();
const ast = parser.parse(ftlSource);

// Manually walking the AST to extract and format messages
for (const entry of ast.body) {
  if (entry.type === "Message" && entry.id.name === "welcome") {
    // Custom resolution logic...
  }
}
```

### Why This Is Wrong

- `@fluent/syntax` produces a full AST — significantly heavier than the runtime parser
- It is designed for **tooling**: linting, programmatic generation, editor integrations
- It does NOT resolve placeables, terms, selectors, or format numbers/dates
- You would need to reimplement the entire Fluent resolver manually
- `@fluent/bundle` includes its own optimized runtime parser that is smaller and faster

### Correct

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftlSource));

const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "Anna" });
}
```

### When @fluent/syntax IS Appropriate

| Use Case | Correct Package |
|----------|----------------|
| Runtime message formatting | `@fluent/bundle` |
| Linting FTL files in CI | `@fluent/syntax` |
| Building an FTL editor | `@fluent/syntax` |
| Extracting message IDs programmatically | `@fluent/syntax` |
| Converting FTL to/from other formats | `@fluent/syntax` |
| Validating FTL before deployment | `@fluent/syntax` |

---

## AP-3: Assuming All Messages Loaded After addResource()

### Wrong

```typescript
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftlSource));

// Assuming every message from the FTL source is available
const msg = bundle.getMessage("some-message")!;
const text = bundle.formatPattern(msg.value!, args);
// Crashes if "some-message" had a parse error
```

### Why This Is Dangerous

- Parse errors are per-message — one broken message does not prevent others from loading
- But the broken message IS missing from the bundle
- Using `!` (non-null assertion) bypasses TypeScript's safety checks
- `getMessage()` returns `undefined` for missing messages
- `msg.value` can be `null` for attribute-only messages

### Correct

```typescript
const bundle = new FluentBundle("en-US");
const errors = bundle.addResource(new FluentResource(ftlSource));

if (errors.length > 0) {
  console.warn("Some messages failed to parse:", errors);
}

const msg = bundle.getMessage("some-message");
if (msg?.value) {
  const formatErrors: Error[] = [];
  const text = bundle.formatPattern(msg.value, args, formatErrors);
  if (formatErrors.length > 0) {
    console.warn("Formatting issues:", formatErrors);
  }
}
```

---

## AP-4: Not Validating FTL Files Before Deployment

### Wrong

```
# Deployment pipeline
1. Write FTL files
2. Commit to repository
3. Deploy to production
   → Parse errors discovered by end users
```

### Why This Is Dangerous

- FTL parse errors are silent in `@fluent/bundle` — broken messages just disappear
- Users see message IDs instead of translations (e.g., `welcome-message` instead of "Welcome!")
- Tab/space errors are invisible in most editors without proper configuration
- Missing default variants look correct to the human eye

### Correct — Add CI Validation

```
# Deployment pipeline
1. Write FTL files
2. Run @fluent/syntax validation in CI
3. Block deployment if Junk entries found
4. Deploy to production
```

```typescript
// validate-ftl.ts — run in CI
import { FluentParser } from "@fluent/syntax";
import { readFileSync, readdirSync } from "fs";

const parser = new FluentParser({ withSpans: true });
let exitCode = 0;

for (const file of readdirSync("locales").filter(f => f.endsWith(".ftl"))) {
  const source = readFileSync(`locales/${file}`, "utf-8");
  const resource = parser.parse(source);

  for (const entry of resource.body) {
    if (entry.type === "Junk") {
      exitCode = 1;
      for (const ann of entry.annotations) {
        const line = source.substring(0, ann.span!.start).split("\n").length;
        console.error(`${file}:${line}: [${ann.code}] ${ann.message}`);
      }
    }
  }
}

process.exit(exitCode);
```

---

## AP-5: Mixing Tab and Space Indentation

### Wrong

```ftl
# Some lines use spaces, others use tabs — visually identical in some editors
message =
    First line with spaces.
→   Second line with tab.
    Third line with spaces.
```

### Why This Is Dangerous

- Tabs and spaces look identical in many editors
- The parser treats tabs as literal text, not indentation
- The tab-indented line breaks the multiline pattern
- Some lines work, others silently fail — extremely hard to debug

### Correct — Configure Editor

ALWAYS set these editor settings for `.ftl` files:

**VS Code** (`settings.json`):
```json
{
  "[ftl]": {
    "editor.insertSpaces": true,
    "editor.tabSize": 4,
    "editor.renderWhitespace": "all"
  }
}
```

**EditorConfig** (`.editorconfig`):
```ini
[*.ftl]
indent_style = space
indent_size = 4
```

---

## AP-6: Silently Falling Back When getMessage Returns Undefined

### Wrong

```typescript
function translate(id: string, args?: Record<string, FluentVariable>): string {
  const msg = bundle.getMessage(id);
  if (!msg?.value) {
    return id; // Silent fallback — hides parse errors
  }
  return bundle.formatPattern(msg.value, args);
}
```

### Why This Is Problematic

- If a message failed to parse, `getMessage()` returns `undefined`
- Silently returning the message ID hides the root cause
- The developer assumes the FTL file is correct because there are no errors
- The broken translation is only discovered when a user reports seeing raw IDs

### Correct — Log the Fallback

```typescript
function translate(id: string, args?: Record<string, FluentVariable>): string {
  const msg = bundle.getMessage(id);
  if (!msg) {
    console.warn(`Missing message: "${id}" — check FTL file for parse errors`);
    return id;
  }
  if (!msg.value) {
    console.warn(`Message "${id}" has no value — only attributes exist`);
    return id;
  }
  const errors: Error[] = [];
  const text = bundle.formatPattern(msg.value, args, errors);
  if (errors.length > 0) {
    console.warn(`Formatting errors for "${id}":`, errors);
  }
  return text;
}
```

---

## AP-7: Using parseEntry() Without Checking for Junk

### Wrong

```typescript
import { FluentParser } from "@fluent/syntax";

const parser = new FluentParser();
const entry = parser.parseEntry("hello = Hello!");
// Assuming entry is always a Message
console.log(entry.id.name); // Works for valid input

const entry2 = parser.parseEntry("123bad = Invalid");
console.log(entry2.id.name); // TypeError — Junk has no .id
```

### Correct

```typescript
const entry = parser.parseEntry(input);
if (entry.type === "Junk") {
  console.error("Parse failed:", entry.annotations[0]?.message);
} else if (entry.type === "Message") {
  console.log("Parsed message:", entry.id.name);
}
```
