# Select Expression Anti-Patterns

## AP-1: Missing Default Variant

### WRONG

```ftl
# BROKEN: No default variant -- syntax error
emails =
    { $count ->
        [one] You have one email.
        [other] You have { $count } emails.
    }
```

### WHY IT FAILS

The Fluent grammar requires exactly one `DefaultVariant` (prefixed with `*`) in every select expression:

```ebnf
variant_list ::= Variant* DefaultVariant Variant* line_end
```

Without `*`, the parser cannot identify a fallback. This produces a parse error and the entire message becomes `Junk`.

### CORRECT

```ftl
emails =
    { $count ->
        [one] You have one email.
       *[other] You have { $count } emails.
    }
```

ALWAYS mark exactly one variant with `*`. Convention is to mark `[other]` as default for plurals.

---

## AP-2: ICU MessageFormat Syntax in FTL

### WRONG

```ftl
# BROKEN: ICU syntax does not work in Fluent
emails = {count, plural, one{You have one email.} other{You have {count} emails.}}
```

### WHY IT FAILS

Fluent is NOT ICU MessageFormat. The `{variable, plural, ...}` syntax is ICU-specific and will be parsed as Junk by the Fluent parser. Fluent uses a completely different select expression syntax.

### CORRECT

```ftl
emails =
    { $count ->
        [one] You have one email.
       *[other] You have { $count } emails.
    }
```

Key differences from ICU:
- Variables use `$` prefix: `$count` not `count`
- Arrow operator `->` instead of comma-separated type
- Variant keys in square brackets `[one]` not curly-brace blocks `one{...}`
- Default marked with `*` instead of implicit

---

## AP-3: Wrong CLDR Categories for the Language

### WRONG

```ftl
# BROKEN: Using only English categories for Polish
pliki =
    { $count ->
        [one] Pobrano { $count } plik.
       *[other] Pobrano { $count } plików.
    }
```

### WHY IT FAILS

Polish has FOUR plural categories: `one`, `few`, `many`, `other`. With only `one` and `other`, values like 2, 3, 4, 22, 23, 24 (which are `few` in Polish) will incorrectly fall through to `*[other]` and display the wrong grammatical form.

### CORRECT

```ftl
pliki =
    { $count ->
        [one] Pobrano { $count } plik.
        [few] Pobrano { $count } pliki.
        [many] Pobrano { $count } plików.
       *[other] Pobrano { $count } plików.
    }
```

ALWAYS check the CLDR plural rules for your target language at:
https://unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html

---

## AP-4: Using [zero] for English

### WRONG

```ftl
# BROKEN: English does not have a CLDR "zero" category
items =
    { $count ->
        [zero] No items.
        [one] One item.
       *[other] { $count } items.
    }
```

### WHY IT FAILS

English CLDR cardinal rules define only `one` (for integer 1) and `other` (for everything else, including 0). The value 0 maps to `other` in English, NOT to `zero`. The `[zero]` variant will NEVER be reached via CLDR matching.

### CORRECT

```ftl
items =
    { $count ->
        [0] No items.
        [one] One item.
       *[other] { $count } items.
    }
```

Use the exact numeric match `[0]` instead of the CLDR category `[zero]`. Exact matches are checked BEFORE CLDR categories, so `[0]` will correctly match when `$count` is 0.

Note: `[zero]` IS valid for languages like Arabic where the CLDR defines a `zero` category.

---

## AP-5: Forgetting $ Prefix on Variable in Selector

### WRONG

```ftl
# BROKEN: "count" without $ is a message reference, not a variable
emails =
    { count ->
        [one] One email.
       *[other] Many emails.
    }
```

### WHY IT FAILS

Without the `$` prefix, `count` is interpreted as a `MessageReference`, not a `VariableReference`. The parser will try to find a message named `count` and use its value as the selector. If no such message exists, the resolution fails silently and the default variant is always used.

### CORRECT

```ftl
emails =
    { $count ->
        [one] One email.
       *[other] Many emails.
    }
```

ALWAYS use `$` prefix for variables passed from the application.

---

## AP-6: Ordinal Plurals Without type: "ordinal"

### WRONG

```ftl
# BROKEN: Without type: "ordinal", cardinal rules apply
ranking = { $pos ->
    [one] {$pos}st
    [two] {$pos}nd
    [few] {$pos}rd
   *[other] {$pos}th
}
```

### WHY IT FAILS

Without `NUMBER($pos, type: "ordinal")`, the selector uses CARDINAL plural rules. In English cardinal rules, `two` and `few` do not exist -- only `one` and `other`. The value 2 maps to cardinal `other`, so "2nd" would display as "2th".

### CORRECT

```ftl
ranking = { NUMBER($pos, type: "ordinal") ->
    [one] {$pos}st
    [two] {$pos}nd
    [few] {$pos}rd
   *[other] {$pos}th
}
```

ALWAYS use `NUMBER($var, type: "ordinal")` when you need ordinal plural categories (1st, 2nd, 3rd, 4th).

---

## AP-7: Multiple Default Variants

### WRONG

```ftl
# BROKEN: Two defaults -- parse error
greeting =
    { $timeOfDay ->
       *[morning] Good morning!
       *[other] Hello!
    }
```

### WHY IT FAILS

The grammar allows exactly ONE `DefaultVariant`. Having two `*`-prefixed variants is a syntax error. The parser will produce Junk.

### CORRECT

```ftl
greeting =
    { $timeOfDay ->
        [morning] Good morning!
       *[other] Hello!
    }
```

Mark exactly ONE variant with `*`. All other variants are regular (non-default).

---

## AP-8: Assuming 0 and 1 Match Categories in All Languages

### WRONG

```ftl
# Assuming 0 is always "other" and 1 is always "one"
fichiers =
    { $count ->
        [one] Un fichier.
       *[other] { $count } fichiers.
    }
# Developer expects: 0 → "0 fichiers" (other)
# Actual in French: 0 → "Un fichier" (one!) — because French maps 0 to "one"
```

### WHY THIS IS MISLEADING

In French, 0 maps to the `one` CLDR category, NOT `other`. A developer testing with English assumptions would expect 0 to display the `other` variant, but French speakers see the `one` variant for zero items.

### CORRECT APPROACH

```ftl
# If you need special handling for zero in French:
fichiers =
    { $count ->
        [0] Aucun fichier.
        [one] Un fichier.
       *[other] { $count } fichiers.
    }
```

Use exact match `[0]` when you need zero-specific text, regardless of the language's CLDR mapping for 0.

---

## AP-9: Nested Selectors Without Inner Defaults

### WRONG

```ftl
# BROKEN: Inner select expressions lack default variants
response =
    { $gender ->
        [masculine] { $count ->
            [one] He sent one message.
            [other] He sent { $count } messages.
        }
       *[other] { $count ->
            [one] They sent one message.
            [other] They sent { $count } messages.
        }
    }
```

### WHY IT FAILS

EVERY select expression -- including nested ones -- MUST have exactly one `*[default]` variant. The inner `{ $count -> ... }` blocks are missing the `*` marker. This causes parse errors.

### CORRECT

```ftl
response =
    { $gender ->
        [masculine] { $count ->
            [one] He sent one message.
           *[other] He sent { $count } messages.
        }
       *[other] { $count ->
            [one] They sent one message.
           *[other] They sent { $count } messages.
        }
    }
```

---

## AP-10: Using Variant Keys That Cannot Match

### WRONG

```ftl
# BROKEN: "1st" is not a valid NumberLiteral or useful Identifier
ranking = { NUMBER($pos, type: "ordinal") ->
    [1st] First place!
    [2nd] Second place!
   *[other] Position { $pos }.
}
```

### WHY IT FAILS

Variant keys must be either a `NumberLiteral` (like `[1]`) or an `Identifier` (like `[one]`). The value `1st` starts with a digit, making it a `NumberLiteral` parse attempt, but `st` is not valid in a number. This causes a parse error.

### CORRECT

```ftl
ranking = { NUMBER($pos, type: "ordinal") ->
    [1] First place!
    [2] Second place!
   *[other] Position { $pos }.
}
```

Use exact numeric matches `[1]`, `[2]` for specific values, and CLDR ordinal categories `[one]`, `[two]`, `[few]`, `[other]` for pattern-based matching.

---

## Official Sources

- https://projectfluent.org/fluent/guide/selectors.html
- https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf
- https://unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html
