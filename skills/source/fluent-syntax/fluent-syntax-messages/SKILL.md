---
name: fluent-syntax-messages
description: "Guides FTL message syntax including simple and multiline messages, placeables with variable references, message references, text elements, whitespace and indentation rules, the dedentation algorithm, three-level comment system, and special character handling. Activates when writing FTL messages, formatting multiline text, using variables in translations, or understanding FTL whitespace rules."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-syntax-messages

## Quick Reference

### Message Anatomy

| Component | Syntax | Example |
|-----------|--------|---------|
| Simple message | `key = Value` | `hello = Hello, world!` |
| Multiline message | Indented continuation lines | See below |
| Block text | Value starts on next line | `key =\n    Block text here` |
| Placeable | `{ expression }` | `{ $user }`, `{ other-msg }` |
| Variable reference | `{ $name }` | `{ $email-count }` |
| Message reference | `{ identifier }` | `{ menu-save }` |

### Identifier Rules

| Rule | Detail |
|------|--------|
| First character | MUST be a letter `[a-zA-Z]` |
| Subsequent characters | Letters, digits, underscores, hyphens `[a-zA-Z0-9_-]` |
| Invalid characters | Dots, spaces, or any other special characters |
| EBNF | `Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*` |

### Comment Levels

| Syntax | Name | Scope | Binding |
|--------|------|-------|---------|
| `#` | Comment | Message-level | Bound to next message if directly above (no blank line) |
| `##` | Group Comment | Section-level | ALWAYS standalone; divides file into groups |
| `###` | Resource Comment | File-level | ALWAYS standalone; describes entire file |

### Critical Warnings

**NEVER use tabs for indentation.** Tabs (U+0009) are NOT whitespace in FTL -- they are treated as regular text characters. ALWAYS use spaces (U+0020) for indentation. This is the number one cause of broken FTL output from AI-generated code.

**NEVER omit the space after `#` in comments.** `#comment` is invalid FTL. It MUST be `# comment`. An empty comment line (`#` alone) IS valid.

**NEVER start a continuation line with `[`, `*`, or `.`** -- these characters are reserved for variant keys, default variant markers, and attribute accessors. Use a quoted text placeable to escape them: `{"["}`.

**NEVER place literal `{` or `}` in text** -- they MUST be escaped using quoted text placeables: `{"{"}` or `{"}"}`.

---

## Simple Messages

The basic unit of translation is a message: an identifier, an equals sign, and a value (called a **pattern**).

```ftl
hello = Hello, world!
app-title = My Application
logout = Log out
```

The text value begins at the first non-blank character after `=`. Leading spaces between `=` and the value are stripped.

---

## Multiline Messages

Text spans multiple lines when each continuation line is indented by at least one space.

```ftl
single = Text can be written in a single line.

multi = Text can also span multiple lines
    as long as each new line is indented
    by at least one space.

block =
    Sometimes it's more readable to format
    multiline text as a "block", starting
    on a new line. All lines must be indented
    by at least one space.
```

### Dedentation Algorithm

In multiline patterns, common indent is stripped from all indented lines:

1. The first line (on the same line as `=`) is EXCLUDED from dedentation calculations.
2. The common indent of all remaining indented lines is calculated.
3. That common indent is removed from every indented line.

```ftl
example =
    This has 4 spaces of indent in source.
        This line retains 4 extra visible spaces.
```

Result: "This has 4 spaces of indent in source.\n    This line retains 4 extra visible spaces."

### Blank Line Handling

| Position | Behavior |
|----------|----------|
| Leading blank lines (between `=` and first content) | Ignored |
| Trailing blank lines (after last content) | Ignored |
| Interior blank lines (content before AND after) | Preserved |

```ftl
leading-spaces =     This value starts with "This" (leading spaces stripped).

leading-lines =


    This value starts with "This".
    The blank lines under the identifier are ignored.

interior-blank =
    First paragraph.

    The blank line above IS preserved in output.
```

---

## Placeables and Variables

Placeables use curly braces `{ }` to incorporate dynamic content into text.

### Variable References

Variables are external data provided by the application at runtime, using `$name` syntax:

```ftl
welcome = Welcome, { $user }!
unread-emails = { $user } has { $email-count } unread emails.
```

Numbers and dates passed as variables are AUTOMATICALLY formatted according to the user's locale. With `$duration = 12345.678`:
- American English: `12,345.678`
- South African English: `12 345,678`

### Message References

One message can embed another by referencing its identifier (without `$`):

```ftl
menu-save = Save
help-menu-save = Click { menu-save } to save the file.
```

This keeps translations consistent and simplifies maintenance.

---

## Comments

### Three-Level System

```ftl
### Localization for Server-side strings of Firefox Screenshots

## Global phrases shared across pages

# This is shown as the page title
my-shots = My Shots
home-link = Home

## Creating page

# Note: { $title } is a placeholder for the title of the web page
# $title (String) - The title of the web page being captured
creating-page-title = Creating { $title }
```

### Binding Rules

- A `#` comment placed **directly above** a message (no blank line) is bound to that message. Localization tools display them together.
- A blank line between a `#` comment and a message makes the comment standalone.
- `##` and `###` comments are ALWAYS standalone and NEVER bind to messages.
- A space MUST follow `#`, `##`, or `###` before comment text.

---

## Special Characters

### Restricted Characters at Line Start

The first character of a continuation line (after indentation) CANNOT be `[`, `*`, or `.`. These are parsed as variant keys, default variant markers, or attribute accessors. ALWAYS escape them:

```ftl
# WRONG -- "[" parsed as variant key start
bracket-wrong =
    First line
    [This breaks]

# CORRECT
bracket-correct =
    First line
    {"["}This works]
```

### Curly Braces in Text

Literal `{` and `}` MUST be escaped:

```ftl
opening-brace = This has a curly brace: {"{"}.
closing-brace = This has a closing brace: {"}"}.
```

### Escape Sequences (Quoted Text Only)

Escape sequences work ONLY inside quoted text (string literals), NOT in regular text:

| Sequence | Meaning |
|----------|---------|
| `\"` | Literal double quote |
| `\\` | Literal backslash |
| `\uHHHH` | Unicode U+0000 to U+FFFF |
| `\UHHHHHH` | Any Unicode code point (6 hex digits) |

```ftl
privacy-label = Privacy{"\u00A0"}Policy
```

In regular (unquoted) text, the backslash `\` has no special meaning.

### Preserving Leading Whitespace

Use a quoted text placeable to force leading spaces in output:

```ftl
indented = {"    "}This message starts with 4 spaces.
```

Without the placeable, the dedentation algorithm strips leading spaces.

---

## Decision Tree: Message Format

```
Need to write a message?
├─ Single line of text → key = Value text here
├─ Multiple lines of text
│  ├─ First line on same line as = → key = First line
│  │                                      continuation here
│  └─ Block format (cleaner) → key =
│                                   All lines indented
│                                   by at least one space
├─ Need dynamic content → Use placeables: { $variable }
├─ Need to reference another message → { other-message }
├─ Need literal { or } → Use {"{"} or {"}"}
└─ Need [ * . at line start → Use {"["} or {"*"} or {"."}
```

---

## EBNF Grammar (Key Productions)

```ebnf
Message          ::= Identifier blank_inline? "=" blank_inline?
                     ((Pattern Attribute*) | (Attribute+))
Pattern          ::= PatternElement+
PatternElement   ::= inline_text | block_text | inline_placeable | block_placeable
Identifier       ::= [a-zA-Z] [a-zA-Z0-9_-]*
VariableReference ::= "$" Identifier
MessageReference ::= Identifier AttributeAccessor?
inline_placeable ::= "{" blank? (SelectExpression | InlineExpression) blank? "}"
CommentLine      ::= ("###" | "##" | "#") ("\u0020" comment_char*)? line_end
blank_inline     ::= "\u0020"+
```

A message MUST have either a value (Pattern), attributes, or both. Identifiers MUST start with a letter and may contain `[a-zA-Z0-9_-]`.

---

## Reference Links

- [references/methods.md](references/methods.md) -- EBNF grammar excerpts for Message, Pattern, Identifier, VariableReference, MessageReference, CommentLine
- [references/examples.md](references/examples.md) -- Complete examples: simple messages, multiline, block text, placeables, variables, message references, comments, special characters
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes: whitespace errors, tab usage, missing comment space, special characters at line start

### Official Sources

- https://projectfluent.org/fluent/guide/hello.html
- https://projectfluent.org/fluent/guide/multiline.html
- https://projectfluent.org/fluent/guide/placeables.html
- https://projectfluent.org/fluent/guide/variables.html
- https://projectfluent.org/fluent/guide/references.html
- https://projectfluent.org/fluent/guide/comments.html
- https://projectfluent.org/fluent/guide/special.html
- https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf
