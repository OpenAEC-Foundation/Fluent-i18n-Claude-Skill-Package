# fluent-errors-parsing — Examples Reference

## Error 1: Tab Indentation

### Wrong

```ftl
# Tab character used for indentation (shown as →)
greeting =
→   Hello, this line uses tab indentation.
→   It will NOT be recognized as a continuation.
```

**Result**: `greeting` becomes Junk. `bundle.getMessage("greeting")` returns `undefined`.

### Correct

```ftl
# Space characters used for indentation
greeting =
    Hello, this line uses space indentation.
    It IS recognized as a continuation.
```

**Result**: `greeting` resolves to `"Hello, this line uses space indentation.\nIt IS recognized as a continuation."`

---

## Error 2: Missing Default Variant

### Wrong

```ftl
emails =
    { $count ->
        [one] You have one email.
        [other] You have { $count } emails.
    }
```

**Result**: Syntax error — no variant is marked with `*` as default. The entire message becomes Junk.

### Correct

```ftl
emails =
    { $count ->
        [one] You have one email.
       *[other] You have { $count } emails.
    }
```

**Result**: Correct — `*[other]` provides the mandatory default variant.

---

## Error 3: Special Character at Line Start

### Wrong — Square Bracket

```ftl
instructions =
    Read the documentation
    [Chapter 1] covers the basics.
```

**Result**: `[Chapter 1]` is parsed as a variant key, breaking the message structure.

### Correct — Square Bracket

```ftl
instructions =
    Read the documentation
    {"["}Chapter 1] covers the basics.
```

### Wrong — Asterisk

```ftl
emphasis =
    This is important
    * very important indeed.
```

**Result**: `*` is parsed as a default variant marker.

### Correct — Asterisk

```ftl
emphasis =
    This is important
    {"*"} very important indeed.
```

### Wrong — Dot

```ftl
sentence =
    End of sentence
    .next" comes after.
```

**Result**: `.next` is parsed as an attribute accessor, breaking the message.

### Correct — Dot

```ftl
sentence =
    End of sentence
    {"."}next comes after.
```

---

## Error 4: Comment Without Space

### Wrong

```ftl
#This comment has no space after the hash.
hello = Hello, world!
```

**Result**: `#This` is not recognized as a valid comment. It becomes Junk.

### Correct

```ftl
# This comment has a space after the hash.
hello = Hello, world!
```

### Also Correct — Empty Comment

```ftl
#
hello = Hello, world!
```

**Result**: A lone `#` on a line is a valid empty comment.

---

## Error 5: Invalid Identifier

### Wrong — Starts with Number

```ftl
404-error = Page not found.
```

**Result**: Identifiers MUST start with a letter. `404-error` is invalid and becomes Junk.

### Correct

```ftl
error-404 = Page not found.
```

### Wrong — Contains Dots

```ftl
app.title = My Application
```

**Result**: Dots are not allowed in identifiers. The parser sees `app` as the identifier and `.title` as an attribute.

### Correct

```ftl
app-title = My Application
```

### Wrong — Contains Spaces

```ftl
my message = Hello
```

**Result**: The parser sees `my` as the identifier and fails to parse `message = Hello` as a value.

### Correct

```ftl
my-message = Hello
```

---

## Error 6: Unescaped Curly Braces

### Wrong

```ftl
code-example = Use { and } in your template.
```

**Result**: The `{` opens a placeable expression. The parser tries to interpret `and` as a message reference, then fails at the unmatched `}`.

### Correct

```ftl
code-example = Use {"{"} and {"}"} in your template.
```

**Result**: `"Use { and } in your template."`

---

## Error 7: Escape in Unquoted Text

### Wrong

```ftl
copyright = \u00A9 2025 Company
```

**Result**: Outputs literal `\u00A9 2025 Company` — the backslash has no special meaning in unquoted text.

### Correct

```ftl
copyright = {"\u00A9"} 2025 Company
```

**Result**: Outputs `(c) 2025 Company` (with actual copyright symbol).

---

## Error 8: No Indentation on Continuation

### Wrong

```ftl
long-text =
This line has no indentation.
It is not recognized as part of the message.
```

**Result**: `long-text` has an empty value. The subsequent lines are either Junk or parsed as new (invalid) entries.

### Correct

```ftl
long-text =
    This line has indentation.
    It is recognized as part of the message.
```

---

## Error 9: Term Without Value

### Wrong

```ftl
-brand-name =
    .gender = feminine
```

**Result**: Terms MUST have a value — unlike messages, a term cannot be attributes-only. This becomes Junk.

### Correct

```ftl
-brand-name = Aurora
    .gender = feminine
```

---

## Full Validation Example with @fluent/syntax

```typescript
import { FluentParser } from "@fluent/syntax";

const ftlSource = `
### My Translations

valid-hello = Hello, world!

# This message has a tab
broken-tab =
\tTab indented line.

missing-default =
    { $count ->
        [one] One.
        [other] Many.
    }

also-valid = Goodbye!
`;

const parser = new FluentParser({ withSpans: true });
const resource = parser.parse(ftlSource);

let errorCount = 0;
for (const entry of resource.body) {
  if (entry.type === "Junk") {
    errorCount++;
    const preview = entry.content.trim().split("\n")[0];
    console.error(`Junk entry: "${preview}"`);

    for (const ann of entry.annotations) {
      const line = ftlSource.substring(0, ann.span!.start).split("\n").length;
      console.error(`  Line ${line}: [${ann.code}] ${ann.message}`);
    }
  }
}

console.log(`Found ${errorCount} parse error(s)`);
console.log(`Total entries: ${resource.body.length}`);
// Junk entries are included in the body alongside valid entries
```

---

## CI Pipeline Validation Script

```typescript
import { readFileSync, readdirSync } from "fs";
import { join } from "path";
import { FluentParser } from "@fluent/syntax";

function validateFtlDirectory(dir: string): boolean {
  const parser = new FluentParser({ withSpans: true });
  let hasErrors = false;

  const files = readdirSync(dir).filter(f => f.endsWith(".ftl"));

  for (const file of files) {
    const path = join(dir, file);
    const source = readFileSync(path, "utf-8");
    const resource = parser.parse(source);

    for (const entry of resource.body) {
      if (entry.type === "Junk") {
        hasErrors = true;
        for (const ann of entry.annotations) {
          const line = source.substring(0, ann.span!.start).split("\n").length;
          console.error(`${file}:${line}: [${ann.code}] ${ann.message}`);
        }
      }
    }
  }

  return !hasErrors;
}

// Usage in CI
const isValid = validateFtlDirectory("./locales/en");
if (!isValid) {
  process.exit(1);
}
```

---

## Comparing Both Parsers on the Same Input

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { FluentParser } from "@fluent/syntax";

const ftl = `
hello = Hello!
broken = {
goodbye = Goodbye!
`;

// @fluent/bundle — runtime parser
const bundle = new FluentBundle("en-US");
const resource = new FluentResource(ftl);
const addErrors = bundle.addResource(resource);
console.log("addResource errors:", addErrors.length);
// → 1 (only "broken" failed)
console.log("hello exists:", bundle.hasMessage("hello"));
// → true
console.log("broken exists:", bundle.hasMessage("broken"));
// → false
console.log("goodbye exists:", bundle.hasMessage("goodbye"));
// → true

// @fluent/syntax — strict parser
const parser = new FluentParser({ withSpans: true });
const ast = parser.parse(ftl);
for (const entry of ast.body) {
  if (entry.type === "Junk") {
    console.log("Junk:", entry.content.trim());
    for (const ann of entry.annotations) {
      console.log(`  ${ann.code}: ${ann.message}`);
    }
  }
}
// → Junk: "broken = {"
// →   E0004: Expected character: "}"
```
