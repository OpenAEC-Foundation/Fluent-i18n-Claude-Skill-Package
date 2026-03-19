# EBNF Grammar Excerpts — FTL Messages

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf
Specification version: Fluent Syntax 1.0

---

## Message

```ebnf
Message ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
```

A Message MUST start with an `Identifier`, followed by optional inline whitespace, an `=` sign, optional inline whitespace, and then EITHER:
- A `Pattern` (value) with zero or more `Attribute` entries, OR
- One or more `Attribute` entries without a value

This means a message can be value-only, value-plus-attributes, or attributes-only.

---

## Pattern

```ebnf
Pattern        ::= PatternElement+
PatternElement ::= inline_text | block_text | inline_placeable | block_placeable
inline_text    ::= text_char+
block_text     ::= blank_block blank_inline indented_char inline_text?
```

A Pattern is one or more `PatternElement` entries. Text can appear inline (on the same line) or as block text (continuation lines). Block text requires a `blank_block` (one or more line endings), followed by `blank_inline` (indentation with spaces), followed by an `indented_char`.

### indented_char

```ebnf
indented_char ::= text_char - "[" - "*" - "."
```

The first character of a continuation line (after indentation) CANNOT be `[`, `*`, or `.`. These characters are reserved for variant keys, default variant markers, and attribute accessors respectively. To use these characters at line start, wrap them in a quoted text placeable: `{"["}`, `{"*"}`, `{"."}`.

---

## Identifier

```ebnf
Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*
```

- MUST start with a letter (a-z or A-Z)
- May contain letters, digits (0-9), underscores (`_`), and hyphens (`-`)
- NO dots, spaces, or other special characters
- Used for message identifiers, term identifiers (after `-` prefix), attribute names, function names, and variable names (after `$` prefix)

---

## VariableReference

```ebnf
VariableReference ::= "$" Identifier
```

A variable reference is a `$` sign followed by an `Identifier`. Variables represent external data provided by the application at runtime. The `$` prefix distinguishes variables from message references.

---

## MessageReference

```ebnf
MessageReference ::= Identifier AttributeAccessor?
AttributeAccessor ::= "." Identifier
```

A message reference is an `Identifier` optionally followed by an `AttributeAccessor` (dot + identifier). When used inside a placeable, it embeds the referenced message's value (or attribute value) into the current pattern.

Examples:
- `{ menu-save }` — references the value of the `menu-save` message
- `{ login-input.placeholder }` — references the `placeholder` attribute of `login-input`

---

## CommentLine

```ebnf
CommentLine  ::= ("###" | "##" | "#") ("\u0020" comment_char*)? line_end
comment_char ::= any_char - line_end
```

A comment line starts with `#`, `##`, or `###`, optionally followed by a single space (U+0020) and comment text, then a line ending. Key rules:

- The space after `#`/`##`/`###` is MANDATORY when comment text follows
- `#comment` (no space) is NOT valid — it MUST be `# comment`
- An empty comment (`#` alone on a line) IS valid
- `##` and `###` comments are ALWAYS standalone (never bound to a message)
- `#` comments directly above a message (no blank line) are bound to that message

---

## Inline Placeable

```ebnf
inline_placeable ::= "{" blank? (SelectExpression | InlineExpression) blank? "}"
InlineExpression ::= StringLiteral
                   | NumberLiteral
                   | FunctionReference
                   | MessageReference
                   | TermReference
                   | VariableReference
                   | inline_placeable
```

A placeable is enclosed in curly braces and contains either a `SelectExpression` or an `InlineExpression`. The `InlineExpression` can be any of the listed types, including nested placeables.

---

## Whitespace Productions

```ebnf
blank_inline ::= "\u0020"+
line_end     ::= "\u000D\u000A" | "\u000A" | EOF
blank_block  ::= (blank_inline? line_end)+
blank        ::= (blank_inline | line_end)+
```

- `blank_inline` is one or more space characters (U+0020 ONLY — tabs are NOT included)
- `line_end` is CRLF, LF, or end of file
- `blank_block` is one or more lines that are empty or contain only spaces
- `blank` is any combination of spaces and line endings

The absence of tab (U+0009) from `blank_inline` is the formal reason why tabs cannot be used for indentation in FTL.
