---
name: fluent-syntax-terms
description: >
  Use when defining or referencing terms (-term-name) in FTL for brand names or shared vocabulary. Prevents accidental runtime API access to terms, which is never allowed.
  Covers term syntax, parameterized terms with named arguments, term attributes for grammatical metadata, and visibility rules.
  Keywords: FTL terms, -term-name, parameterized terms, term attributes, brand names, grammatical cases, runtime visibility.
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-syntax-terms

## Quick Reference

### Term Syntax Overview

| Feature | Syntax | Example |
|---------|--------|---------|
| Define a term | `-name = Value` | `-brand-name = Firefox` |
| Reference a term | `{ -name }` | `about = About { -brand-name }.` |
| Parameterized term | `{ -name(key: "value") }` | `{ -brand-name(case: "locative") }` |
| Term attribute | `-name = Value` + `.attr = data` | `-brand-name = Firefox` + `.gender = masculine` |
| Attribute as selector | `{ -name.attr -> ... }` | `{ -brand-name.gender -> [masculine] ... }` |

### Term vs Message Comparison

| Aspect | Message | Term |
|--------|---------|------|
| Identifier prefix | None (starts with letter) | `-` (hyphen prefix) |
| Runtime access | `bundle.getMessage("id")` returns it | `bundle.getMessage("-id")` returns `undefined` |
| Purpose | User-facing text displayed in UI | Shared vocabulary, brand consistency, grammatical metadata |
| Variable data source | Received from application code | Received from referencing messages via named arguments |
| Value requirement | Can be attribute-only (no value) | MUST have a value (Pattern required) |
| Attribute visibility | Public (accessible via `.attributes`) | Private (usable ONLY as selectors in FTL) |
| Named argument values | N/A (messages use application variables) | MUST be string or number literals |

### Critical Warnings

**NEVER** try to access terms via `bundle.getMessage("-term-name")` -- this returns `undefined`. Terms are resolved automatically when referenced inside messages. There is NO public API to directly format a term.

**NEVER** use variable references as named argument values in term references. `{ -term(case: $myCase) }` is NOT valid FTL. Named argument values MUST be string literals (`"locative"`) or number literals (`1`).

**ALWAYS** give terms a value (Pattern). Unlike messages, a term CANNOT be attribute-only. The grammar requires `Term ::= "-" Identifier "=" Pattern Attribute*` -- the Pattern is mandatory because terms are designed to be interpolated in placeables.

**ALWAYS** use the hyphen prefix when defining terms. Without the hyphen, you are defining a regular message, which changes visibility and variable resolution semantics.

**NEVER** treat term attributes as retrievable data. Term attributes like `.gender = feminine` are private to the FTL file and can ONLY be used as selectors within FTL. They are NOT accessible via any runtime API.

---

## Term Definition and Usage

### Basic Terms

Terms define reusable, internal-only translation fragments. The hyphen prefix (`-`) distinguishes them from messages:

```ftl
-brand-name = Firefox
-app-version = 3.0

about = About { -brand-name }.
update-successful = { -brand-name } has been updated to version { -app-version }.
```

Terms enforce brand consistency: every message referencing `-brand-name` gets the same value. When the brand name changes, update the term once.

### Parameterized Terms

Terms accept named arguments from referencing messages. This enables grammatical case handling in morphologically rich languages:

```ftl
# Polish: brand name declines by case
-brand-name =
    { $case ->
       *[nominative] Firefox
        [locative] Firefoksie
        [genitive] Firefoksa
    }

about = Informacje o { -brand-name(case: "locative") }.
download = Pobierz { -brand-name(case: "genitive") }.
welcome = { -brand-name } jest gotowy.
```

When no arguments are passed (as in `welcome`), the `$case` variable is unset, so the `*[nominative]` default variant is selected.

**Rule**: Named argument values MUST be literals. `{ -brand-name(case: "locative") }` is valid. `{ -brand-name(case: $userCase) }` is NOT valid.

### Parameterized Terms with Dynamic Values

Terms can construct values from parameters:

```ftl
-https = https://{ $host }
visit = Visit { -https(host: "example.com") } for more information.
```

---

## Term Attributes

### Grammatical Metadata

Term attributes store private metadata used for selector-based logic in other messages:

```ftl
-brand-name = Aurora
    .gender = feminine

update-successful =
    { -brand-name.gender ->
        [masculine] { -brand-name } został zaktualizowany.
        [feminine] { -brand-name } została zaktualizowana.
       *[other] Program { -brand-name } został zaktualizowany.
    }
```

The `.gender` attribute on `-brand-name` is used as a selector in `update-successful`. The attribute value itself is NEVER exposed to the application -- it controls FTL-internal branching only.

### Multiple Attributes

A term can have multiple attributes for different grammatical properties:

```ftl
-product-name = Thunderbird
    .gender = masculine
    .animacy = inanimate
    .starts-with-vowel = no
```

Each attribute is available as a selector via `-product-name.gender`, `-product-name.animacy`, etc.

---

## Decision Tree

### When to Use a Term vs a Message

```
Is this text displayed directly to the user as a standalone string?
├── YES → Use a message (no hyphen prefix)
└── NO → Is it referenced by other messages for consistency?
    ├── YES → Use a term (hyphen prefix)
    └── NO → Is it grammatical metadata (gender, case forms)?
        ├── YES → Use a term with attributes
        └── NO → It may not need to exist at all
```

### When to Use Parameterized Terms

```
Does the term need different forms based on grammatical context?
├── YES → Use a parameterized term with named arguments
│   Example: -brand-name with $case selector
└── NO → Does the term need dynamic construction?
    ├── YES → Use a parameterized term with value parameters
    │   Example: -https with $host parameter
    └── NO → Use a simple term (-brand-name = Firefox)
```

### When to Use Term Attributes

```
Do other messages need to branch based on a property of this term?
├── YES → Add an attribute to the term (.gender = feminine)
│   Then use as selector: { -term.attribute -> ... }
└── NO → Do not add attributes; keep the term simple
```

---

## EBNF Grammar

```ebnf
Term          ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
TermReference ::= "-" Identifier AttributeAccessor? CallArguments?
CallArguments ::= blank? "(" blank? argument_list blank? ")"
NamedArgument ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
```

Key constraints from the grammar:

1. **Term MUST have a value** -- `Pattern` is required (not optional like in Message)
2. **TermReference can combine attribute access and call arguments** -- `{ -term.attr(key: "val") }` is syntactically valid
3. **Named argument values are restricted** -- ONLY `StringLiteral` or `NumberLiteral`, NEVER variable references

---

## Reference Links

- [references/methods.md](references/methods.md) -- EBNF grammar details, term vs message grammar comparison, named argument restrictions
- [references/examples.md](references/examples.md) -- Complete working examples: basic terms, parameterized terms, term attributes, selectors
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do with terms, with explanations

### Official Sources

- https://projectfluent.org/fluent/guide/terms.html
- https://github.com/projectfluent/fluent/blob/master/spec/fluent.ebnf
- https://github.com/projectfluent/fluent.js/tree/main/fluent-bundle
