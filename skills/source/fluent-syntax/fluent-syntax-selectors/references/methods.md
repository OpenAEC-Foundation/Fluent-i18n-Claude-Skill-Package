# Select Expression Grammar and CLDR Reference

## EBNF Grammar: Select Expressions

The following productions define select expressions in Fluent Syntax 1.0.

### SelectExpression

```ebnf
SelectExpression ::= InlineExpression blank? "->" blank_inline? variant_list
```

A select expression consists of an inline expression (the selector), followed by the arrow operator `->`, followed by a variant list. The selector is evaluated at runtime to determine which variant to use.

### variant_list

```ebnf
variant_list ::= Variant* DefaultVariant Variant* line_end
```

The variant list MUST contain exactly ONE `DefaultVariant`. Additional `Variant` entries can appear before and/or after the default. The grammar enforces this -- a select expression without a default variant is unparseable.

### Variant

```ebnf
Variant ::= line_end blank? VariantKey blank_inline? Pattern
```

Each variant starts on a new line, optionally indented, followed by a variant key in square brackets and a pattern (the translation text).

### DefaultVariant

```ebnf
DefaultVariant ::= line_end blank? "*" VariantKey blank_inline? Pattern
```

The default variant is identical to a regular variant except for the `*` prefix before the variant key. This variant is selected when no other variant key matches.

### VariantKey

```ebnf
VariantKey ::= "[" blank? (NumberLiteral | Identifier) blank? "]"
```

A variant key is enclosed in square brackets and contains either:
- A `NumberLiteral` -- for exact numeric matching (e.g., `[0]`, `[1]`, `[-1]`, `[3.14]`)
- An `Identifier` -- for CLDR plural category matching (e.g., `[one]`, `[other]`) or string matching

### Supporting Productions

```ebnf
NumberLiteral    ::= "-"? digits ("." digits)?
Identifier       ::= [a-zA-Z] [a-zA-Z0-9_-]*
digits           ::= [0-9]+
InlineExpression ::= StringLiteral | NumberLiteral | FunctionReference
                   | MessageReference | TermReference | VariableReference
                   | inline_placeable
```

### Key Grammar Observations

1. The `*` marker can appear at ANY position in the variant list (not only last). Placing it last is a convention, not a requirement.
2. `NumberLiteral` supports negative numbers and decimals, so `[-1]` and `[0.5]` are valid variant keys.
3. `Identifier` must start with a letter, so `[1st]` is NOT a valid identifier key -- it would need to be a number literal or written differently.
4. Whitespace inside brackets is allowed: `[ one ]` is equivalent to `[one]`.

---

## CLDR Plural Categories: Complete Reference

### All Possible Categories

The Unicode CLDR defines exactly six plural categories. No language uses categories outside this set.

| Category | Meaning | Example Languages |
|----------|---------|-------------------|
| `zero` | Zero quantity form | Arabic, Latvian, Welsh |
| `one` | Singular / single item | English, French, German, Spanish, Polish, Russian |
| `two` | Dual form | Arabic, Hebrew, Slovenian |
| `few` | Paucal / small count | Polish, Czech, Russian, Arabic, Croatian |
| `many` | Large count form | Polish, Russian, Arabic, Ukrainian |
| `other` | General / default form | ALL languages (every language has `other`) |

### Cardinal Plural Rules by Language

#### English
| Category | Integer Examples | Rule |
|----------|-----------------|------|
| `one` | 1 | Exactly 1 (integer) |
| `other` | 0, 2, 3, 4, 5, 6, ... | Everything else |

#### French
| Category | Integer Examples | Rule |
|----------|-----------------|------|
| `one` | 0, 1 | 0 or 1 |
| `other` | 2, 3, 4, 5, 6, ... | 2 and above |

Note: French treats 0 as `one`, unlike English where 0 is `other`.

#### German
| Category | Integer Examples | Rule |
|----------|-----------------|------|
| `one` | 1 | Exactly 1 |
| `other` | 0, 2, 3, 4, 5, ... | Everything else |

#### Spanish
| Category | Integer Examples | Rule |
|----------|-----------------|------|
| `one` | 1 | Exactly 1 |
| `other` | 0, 2, 3, 4, 5, ... | Everything else |

#### Polish
| Category | Integer Examples | Rule |
|----------|-----------------|------|
| `one` | 1 | Exactly 1 |
| `few` | 2, 3, 4, 22, 23, 24, 32-34, ... | Ends in 2-4, but not 12-14 |
| `many` | 0, 5-21, 25-31, 35-41, ... | Ends in 0, 1, or 5-9; or 12-14 |
| `other` | 1.5, 2.5, ... | Non-integer values |

#### Russian
| Category | Integer Examples | Rule |
|----------|-----------------|------|
| `one` | 1, 21, 31, 41, 51, 61, ... | Ends in 1, but not 11 |
| `few` | 2, 3, 4, 22, 23, 24, 32-34, ... | Ends in 2-4, but not 12-14 |
| `many` | 0, 5-20, 25-30, 35-40, ... | Ends in 0, 5-9, or 11-14 |
| `other` | 1.5, 2.5, ... | Non-integer values |

#### Arabic
| Category | Integer Examples | Rule |
|----------|-----------------|------|
| `zero` | 0 | Exactly 0 |
| `one` | 1 | Exactly 1 |
| `two` | 2 | Exactly 2 |
| `few` | 3-10, 103-110, ... | 3-10 and numbers ending in 03-10 |
| `many` | 11-99, 111-199, ... | 11-99 and numbers ending in 11-99 |
| `other` | 100-102, 200-202, ... | Everything else |

#### Japanese / Chinese / Korean
| Category | Integer Examples | Rule |
|----------|-----------------|------|
| `other` | ALL | No plural distinction -- only `other` exists |

### Ordinal Plural Rules (English)

Used with `NUMBER($var, type: "ordinal")`:

| Category | Integer Examples | Suffix |
|----------|-----------------|--------|
| `one` | 1, 21, 31, 41, 51, ... | -st (1st, 21st) |
| `two` | 2, 22, 32, 42, 52, ... | -nd (2nd, 22nd) |
| `few` | 3, 23, 33, 43, 53, ... | -rd (3rd, 23rd) |
| `other` | 4-20, 24-30, 34-40, ... | -th (4th, 11th, 12th, 13th) |

Note: 11, 12, 13 are `other` (11th, 12th, 13th), NOT `one`, `two`, `few`.

---

## Variant Key Resolution Order

When a select expression is evaluated, keys are checked in this order:

1. **Exact numeric match** -- If the selector value is a number and a variant key is a `NumberLiteral` that exactly equals the value, it matches first.
2. **CLDR category match** -- The number is mapped to its CLDR plural category for the current locale, and that category identifier is matched against variant keys.
3. **Default variant** -- If no exact or category match is found, the `*`-prefixed default variant is used.

This means `[0]` (exact match) takes priority over `[zero]` (CLDR category) even in languages like Arabic where 0 has a `zero` category.

---

## Official Sources

- Fluent selector guide: https://projectfluent.org/fluent/guide/selectors.html
- Fluent EBNF specification: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf
- CLDR plural rules: https://unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html
- NUMBER function: https://projectfluent.org/fluent/guide/builtins.html
