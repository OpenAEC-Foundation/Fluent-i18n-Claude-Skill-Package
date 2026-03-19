---
name: fluent-errors-parsing
description: "Diagnoses and resolves FTL parse errors including indentation mistakes with tabs vs spaces, missing default variant in selectors, special character restrictions at line start, comment space requirement, identifier format violations, junk entry recovery, and @fluent/syntax FluentParser for strict validation. Activates when encountering FTL syntax errors, debugging broken translations, or validating FTL files."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0, @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-errors-parsing

## Quick Reference

### Parse Error Detection Points

| Detection Point | Package | Returns |
|----------------|---------|---------|
| `new FluentResource(ftl)` | `@fluent/bundle` | Silent — invalid entries become inaccessible |
| `bundle.addResource(resource)` | `@fluent/bundle` | `Array<Error>` — one error per broken message |
| `new FluentParser().parse(ftl)` | `@fluent/syntax` | Full AST with `Junk` nodes and `Annotation` details |

### Critical Warnings

**NEVER** use tabs for indentation in FTL files. Tabs (U+0009) are treated as literal text, NOT whitespace. ALWAYS use spaces (U+0020). A tab-indented continuation line will NOT be recognized as part of the message — it becomes Junk.

**NEVER** ignore the error array returned by `addResource()`. Parse errors are per-message and non-blocking — valid messages still load, but broken ones silently disappear without error checking.

**NEVER** use `@fluent/syntax` as a runtime parser for formatting messages. It produces a full AST for tooling purposes. ALWAYS use `@fluent/bundle` (with `FluentResource`) for runtime formatting.

**ALWAYS** include `*[other]` as the default variant in every select expression. Omitting it is a syntax error that produces Junk.

**ALWAYS** put a space after `#` in comments. `#comment` is invalid; `# comment` is correct. An empty `#` on its own line IS valid.

---

## Diagnostic Table: Common Parse Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Message silently missing | Tab used for indentation | Replace ALL tabs with spaces |
| Continuation line not recognized | No leading space on continuation | Indent continuation lines by at least 1 space |
| Select expression becomes Junk | No `*[other]` default variant | Add `*[other]` to every selector |
| Continuation line parsed as new entry | Line starts with `[`, `*`, or `.` | Wrap in quoted placeable: `{"["}` |
| Comment becomes Junk | No space after `#` | Use `# text` not `#text` |
| Message ID rejected | Starts with digit or contains invalid chars | Start with letter, use only `[a-zA-Z0-9_-]` |
| Curly brace causes parse failure | Literal `{` or `}` in text | Escape as `{"{"}` or `{"}"}` |
| Backslash not escaping | Escape used in unquoted text | Escapes work ONLY inside `{"..."}` quoted strings |
| Junk entry in AST | Any unrecoverable syntax error | Parser skips to next entry boundary |
| Annotation on Junk node | `@fluent/syntax` detected specific error | Read `annotation.code` and `annotation.message` |

---

## Tab vs. Space: The #1 FTL Parse Error

FTL treats ONLY U+0020 (space) as indentation whitespace. Tabs (U+0009) are regular text characters.

```ftl
# WRONG — tabs look like indentation but are treated as text
hello =
→   This line uses a tab indent.
→   The parser does NOT see this as a continuation.

# CORRECT — spaces for indentation
hello =
    This line uses space indent.
    The parser recognizes this as a continuation.
```

(Above: `→` represents a tab character. In actual FTL files, tab-indented lines produce Junk or are silently dropped.)

### How Tab Errors Manifest

1. `FluentResource` parser silently drops the message — `bundle.getMessage("hello")` returns `undefined`
2. `addResource()` returns an error for the affected message
3. `FluentParser` from `@fluent/syntax` produces a `Junk` node with an `Annotation`

### Editor Configuration

ALWAYS configure your editor for FTL files:
- Set `insertSpaces: true`
- Set `tabSize: 2` or `tabSize: 4`
- Enable "Render Whitespace" to visually distinguish tabs from spaces

---

## Decision Tree: Diagnosing a Parse Error

```
Message not rendering?
├── Does bundle.getMessage(id) return undefined?
│   ├── YES → Message failed to parse
│   │   ├── Check addResource() error array
│   │   ├── Check for tab characters in indentation
│   │   ├── Check identifier format: starts with letter, [a-zA-Z0-9_-] only
│   │   └── Check for unescaped { } in text
│   └── NO → Message exists but formatting fails
│       └── See fluent-errors-resolution skill
├── Is the message partially broken?
│   ├── Check select expression has *[other] default
│   ├── Check continuation lines start with space (not [, *, or .)
│   └── Check comment format: # followed by space
└── Need exact error location?
    └── Use @fluent/syntax FluentParser with withSpans: true
```

---

## addResource() Error Handling

`addResource()` returns an `Array<Error>` containing one error per message that failed to parse. Other valid messages in the same resource are still added successfully.

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
const resource = new FluentResource(`
valid-message = This works fine.
broken =
	Tab-indented line causes failure.
also-valid = This also works fine.
`);

const errors = bundle.addResource(resource);
// errors.length === 1 (only "broken" failed)
// bundle.hasMessage("valid-message") === true
// bundle.hasMessage("broken") === false
// bundle.hasMessage("also-valid") === true

if (errors.length > 0) {
  for (const err of errors) {
    console.error("Parse error:", err.message);
  }
}
```

**ALWAYS** check the returned error array. NEVER assume all messages loaded successfully.

---

## Junk Recovery: How the Parser Stays Resilient

When the parser encounters syntax it cannot understand, it produces a `Junk` entry and advances to the next line that starts with `#`, `-`, or a letter (potential entry boundaries). This means one broken message does NOT break the entire file.

From the EBNF grammar:
```
Junk      ::= junk_line (junk_line - "#" - "-" - [a-zA-Z])*
junk_line ::= /[^\n]*/ ("\u000A" | EOF)
```

Junk continues consuming lines until it finds a line that could start a new entry. This is why proper message separation with blank lines is important — it helps the parser recover faster.

---

## @fluent/syntax: Strict Validation

For build-time validation, linting, or CI pipelines, use `@fluent/syntax` instead of `@fluent/bundle`. It provides exact error positions via `Span` and detailed error codes via `Annotation`.

```typescript
import { FluentParser } from "@fluent/syntax";

const parser = new FluentParser({ withSpans: true });
const resource = parser.parse(`
hello = Hello, world!
broken = {
missing-default =
    { $count ->
        [one] One item.
    }
`);

for (const entry of resource.body) {
  if (entry.type === "Junk") {
    console.error("Junk content:", entry.content);
    for (const annotation of entry.annotations) {
      console.error(
        `  Error ${annotation.code}: ${annotation.message}`,
        `at position ${annotation.span?.start}-${annotation.span?.end}`
      );
    }
  }
}
```

### When to Use Which Parser

| Scenario | Use |
|----------|-----|
| Runtime message formatting | `@fluent/bundle` (`FluentResource`) |
| CI/CD FTL validation | `@fluent/syntax` (`FluentParser`) |
| Editor tooling / linting | `@fluent/syntax` (`FluentParser` with `withSpans: true`) |
| Programmatic FTL generation | `@fluent/syntax` (`FluentSerializer`) |
| Extracting message IDs | `@fluent/syntax` (`FluentParser`) |

---

## Escape Sequences: Quoted Text Only

Escape sequences (`\"`, `\\`, `\uHHHH`, `\UHHHHHH`) work ONLY inside quoted string literals, NOT in regular text.

```ftl
# WRONG — backslash has no special meaning in unquoted text
privacy = Privacy\u00A0Policy
# Output: Privacy\u00A0Policy (literal backslash characters)

# CORRECT — escape inside quoted text placeable
privacy = Privacy{"\u00A0"}Policy
# Output: Privacy Policy (with non-breaking space)
```

### Valid Escape Sequences (Inside Quoted Text Only)

| Sequence | Result |
|----------|--------|
| `\"` | Literal double quote |
| `\\` | Literal backslash |
| `\uHHHH` | Unicode U+0000 to U+FFFF |
| `\UHHHHHH` | Any Unicode code point (6 hex digits) |

---

## Special Characters at Line Start

The first character after indentation on a continuation line CANNOT be `[`, `*`, or `.` — these are reserved syntax characters (variant key, default variant marker, attribute accessor).

```ftl
# WRONG — [ at line start is parsed as variant key
note =
    Please read
    [the manual] for details.

# CORRECT — wrap in quoted placeable
note =
    Please read
    {"["}the manual] for details.
```

The same applies to `*` and `.`:
```ftl
# WRONG
stars =
    Rate us:
    *** five stars ***

# CORRECT
stars =
    Rate us:
    {"*"}** five stars ***
```

---

## Identifier Format Rules

Message and term identifiers follow strict rules from the EBNF grammar:

```
Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*
```

| Identifier | Valid? | Reason |
|-----------|--------|--------|
| `hello` | Yes | Starts with letter |
| `hello-world` | Yes | Hyphens allowed |
| `hello_world` | Yes | Underscores allowed |
| `Hello123` | Yes | Digits allowed after first char |
| `123hello` | No | MUST start with letter |
| `hello.world` | No | Dots not allowed in identifiers |
| `hello world` | No | Spaces not allowed |
| `héllo` | No | Only ASCII letters `[a-zA-Z]` |

---

## Reference Links

- [references/methods.md](references/methods.md) -- addResource error return, FluentParser API, Junk and Annotation types
- [references/examples.md](references/examples.md) -- Common parse errors with wrong and correct FTL, @fluent/syntax validation
- [references/anti-patterns.md](references/anti-patterns.md) -- Ignoring parse errors, using @fluent/syntax for runtime

### Official Sources

- https://projectfluent.org/fluent/guide/
- https://github.com/projectfluent/fluent.js/tree/main/fluent-bundle
- https://github.com/projectfluent/fluent.js/tree/main/fluent-syntax
- https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf
