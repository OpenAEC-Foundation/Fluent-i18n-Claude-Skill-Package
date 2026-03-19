---
name: fluent-syntax-selectors
description: >
  Use when writing select expressions, plural forms, or variant-based translations in FTL. Prevents missing default variants and incorrect CLDR plural category usage.
  Covers cardinal and ordinal plurals, exact numeric matches, string selectors, and nested selectors.
  Keywords: select expression, CLDR plural categories, variants, default variant, cardinal, ordinal, gender selector.
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-syntax-selectors

## Quick Reference

### Select Expression Anatomy

```ftl
message-id =
    { $variable ->
        [key1] Pattern for key1.
        [key2] Pattern for key2.
       *[default] Fallback pattern.
    }
```

| Component | Syntax | Role |
|-----------|--------|------|
| Selector | `$variable ->` | Runtime value to match against |
| Variant | `[key] pattern` | One possible translation |
| Default variant | `*[key] pattern` | MANDATORY fallback when no key matches |
| Variant key | `[identifier]` or `[number]` | String or numeric match target |

### Selector Types

| Type | Selector Expression | Variant Keys | Example |
|------|-------------------|--------------|---------|
| Cardinal plural | `$count ->` | CLDR categories (`one`, `other`, ...) | `[one] ... *[other] ...` |
| Ordinal plural | `NUMBER($pos, type: "ordinal") ->` | CLDR ordinal categories | `[one] {$pos}st *[other] {$pos}th` |
| Exact numeric | `$count ->` | Number literals | `[0] None. [1] One. *[other] Many.` |
| Formatted number | `NUMBER($val, minimumFractionDigits: 1) ->` | Formatted literals | `[0.0] Zero. *[other] ...` |
| String | `$gender ->` | String identifiers | `[masculine] ... *[other] ...` |
| Nested | Multiple `->` levels | Any combination | See examples |

### CLDR Plural Categories by Language

| Language | Cardinal Categories | Example for `one` |
|----------|--------------------|--------------------|
| English | `one`, `other` | 1 item |
| French | `one`, `other` | 0, 1 (0 and 1 are both `one`) |
| Arabic | `zero`, `one`, `two`, `few`, `many`, `other` | 1 |
| Polish | `one`, `few`, `many`, `other` | 1 |
| Japanese | `other` | (no plural distinction) |
| Russian | `one`, `few`, `many`, `other` | 1, 21, 31... |

All six possible CLDR categories: `zero`, `one`, `two`, `few`, `many`, `other`.

### Critical Warnings

**EVERY** select expression MUST have exactly one `*[default]` variant. Omitting it is a syntax error. The grammar enforces this -- there is no workaround.

**NEVER** use ICU MessageFormat syntax `{count, plural, one{...} other{...}}` in FTL files. Fluent uses `{ $count -> [one] ... *[other] ... }`.

**ALWAYS** include `*[other]` as the default variant for plural selectors. The `other` category exists in every language and is the safest default.

**NEVER** assume English plural categories apply to other languages. English uses only `one` and `other`. Polish uses `one`, `few`, `many`, `other`. Arabic uses all six categories.

**ALWAYS** remember that exact numeric matches (`[0]`, `[1]`) are checked BEFORE CLDR category matches. Use exact matches to special-case specific values.

---

## Decision Trees

### Which Selector Type Do I Need?

```
Is the selector based on a number?
├── YES: Does it need ordinal suffixes (1st, 2nd, 3rd)?
│   ├── YES → Use NUMBER($var, type: "ordinal") selector
│   └── NO: Do you need special formatting (decimal places)?
│       ├── YES → Use NUMBER($var, minimumFractionDigits: N) selector
│       └── NO: Do you need exact value matches (0, 1, 42)?
│           ├── YES → Combine exact [N] keys with CLDR *[other]
│           └── NO → Use plain $var selector with CLDR categories
└── NO: Is it matching a string value?
    └── YES → Use $var selector with string variant keys
```

### How Many CLDR Categories Do I Need?

```
What language are you localizing for?
├── Japanese/Chinese/Korean → Only *[other] (no plural forms)
├── English/German/Dutch → [one] and *[other]
├── French/Portuguese (BR) → [one] and *[other] (note: 0 is "one" in French)
├── Polish/Czech/Slovak → [one], [few], [many], *[other]
├── Arabic → [zero], [one], [two], [few], [many], *[other]
└── Unsure → Check CLDR plural rules, ALWAYS include *[other]
```

---

## Select Expression Patterns

### Cardinal Plurals (Basic)

```ftl
emails =
    { $count ->
        [one] You have one unread email.
       *[other] You have { $count } unread emails.
    }
```

The runtime evaluates `$count` against CLDR cardinal plural rules for the current locale. For English: 1 matches `[one]`, everything else matches `*[other]`.

### Cardinal Plurals (Multi-Category Language)

```ftl
# Polish: one, few, many, other
emails =
    { $count ->
        [one] Masz jedną nieprzeczytaną wiadomość.
        [few] Masz { $count } nieprzeczytane wiadomości.
        [many] Masz { $count } nieprzeczytanych wiadomości.
       *[other] Masz { $count } nieprzeczytanych wiadomości.
    }
```

### Exact Numeric Matches with CLDR Fallback

```ftl
items-in-cart =
    { $count ->
        [0] Your cart is empty.
        [1] You have one item in your cart.
       *[other] You have { $count } items in your cart.
    }
```

Exact matches `[0]` and `[1]` are checked FIRST. If `$count` is 0, the `[0]` variant wins even though 0 would match `*[other]` by CLDR rules. If `$count` is 5, no exact match exists so CLDR resolves to `*[other]`.

### Ordinal Plurals

```ftl
your-rank = { NUMBER($pos, type: "ordinal") ->
    [1] You finished first!
    [one] You finished {$pos}st
    [two] You finished {$pos}nd
    [few] You finished {$pos}rd
   *[other] You finished {$pos}th
}
```

The `type: "ordinal"` argument switches to ordinal CLDR categories. Exact match `[1]` is checked first. Then CLDR ordinal rules apply: in English, `one` = 1st/21st/31st, `two` = 2nd/22nd, `few` = 3rd/23rd, `other` = 4th-20th/24th-30th, etc.

### String Selectors

```ftl
# $gender is a string passed from the application
welcome =
    { $gender ->
        [masculine] Welcome, he is here.
        [feminine] Welcome, she is here.
       *[other] Welcome, they are here.
    }
```

String selectors match variant keys as literal string comparisons. The application passes `$gender` as `"masculine"`, `"feminine"`, or any other string. Non-matching values fall through to `*[other]`.

### Formatted Number Selectors

```ftl
your-score =
    { NUMBER($score, minimumFractionDigits: 1) ->
        [0.0] You scored zero points. What happened?
       *[other] You scored { NUMBER($score, minimumFractionDigits: 1) } points.
    }
```

The NUMBER function with formatting options controls how the value is rendered AND which variant it matches. The formatted representation `0.0` is matched against `[0.0]`.

### Nested Selectors

```ftl
response =
    { $gender ->
        [masculine] { $count ->
            [one] He wrote one message.
           *[other] He wrote { $count } messages.
        }
        [feminine] { $count ->
            [one] She wrote one message.
           *[other] She wrote { $count } messages.
        }
       *[other] { $count ->
            [one] They wrote one message.
           *[other] They wrote { $count } messages.
        }
    }
```

Nested selectors allow multi-dimensional variant selection. Each outer variant contains a complete inner select expression. Every inner expression MUST also have its own `*[default]` variant.

---

## EBNF Grammar

```ebnf
SelectExpression ::= InlineExpression blank? "->" blank_inline? variant_list
variant_list     ::= Variant* DefaultVariant Variant* line_end
Variant          ::= line_end blank? VariantKey blank_inline? Pattern
DefaultVariant   ::= line_end blank? "*" VariantKey blank_inline? Pattern
VariantKey       ::= "[" blank? (NumberLiteral | Identifier) blank? "]"
```

Key rules enforced by this grammar:

1. `variant_list` contains exactly ONE `DefaultVariant` (the `*`-prefixed variant).
2. The default variant can appear at ANY position in the list, not only last.
3. Variant keys are either `NumberLiteral` (for exact matches) or `Identifier` (for CLDR categories or string matching).
4. A `NumberLiteral` in a variant key can be negative: `-1` is valid.

---

## Common Mistakes

| Mistake | Why It Fails | Fix |
|---------|-------------|-----|
| Missing `*` on default variant | Syntax error: grammar requires exactly one DefaultVariant | Add `*` before the fallback variant key |
| Using ICU syntax `{count, plural, ...}` | Fluent does not support ICU MessageFormat | Rewrite as `{ $count -> [one] ... *[other] ... }` |
| Using English categories for Polish | Polish needs `few` and `many`, not just `one`/`other` | Check CLDR plural rules for the target language |
| Forgetting `$` on variable in selector | Without `$`, it is a message reference, not a variable | ALWAYS prefix variables with `$` |
| Using `[zero]` for English 0 | English CLDR `zero` category does not exist; 0 maps to `other` | Use exact match `[0]` instead of `[zero]` |
| Ordinal without `type: "ordinal"` | Cardinal rules apply by default; ordinal suffixes will be wrong | Use `NUMBER($var, type: "ordinal")` |

---

## Reference Links

- [references/methods.md](references/methods.md) -- EBNF grammar details, CLDR category table per language
- [references/examples.md](references/examples.md) -- Complete working examples for all selector types
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do, with explanations

### Official Sources

- https://projectfluent.org/fluent/guide/selectors.html
- https://projectfluent.org/fluent/guide/builtins.html
- https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf
- https://unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html
