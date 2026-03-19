# Select Expression Examples

## Cardinal Plurals

### English (two categories: one, other)

```ftl
# Basic plural: one vs. other
emails =
    { $count ->
        [one] You have one unread email.
       *[other] You have { $count } unread emails.
    }
```

Runtime results:
- `$count = 1` → "You have one unread email."
- `$count = 0` → "You have 0 unread emails."
- `$count = 5` → "You have 5 unread emails."
- `$count = 1000` → "You have 1,000 unread emails." (auto-formatted)

### French (zero and one are both "one" category)

```ftl
# French: 0 and 1 both match [one]
fichiers =
    { $count ->
        [one] { $count } fichier téléchargé.
       *[other] { $count } fichiers téléchargés.
    }
```

Runtime results:
- `$count = 0` → "0 fichier téléchargé." (0 is `one` in French)
- `$count = 1` → "1 fichier téléchargé."
- `$count = 2` → "2 fichiers téléchargés."

### Polish (four categories: one, few, many, other)

```ftl
# Polish: complex plural rules
pliki =
    { $count ->
        [one] Pobrano { $count } plik.
        [few] Pobrano { $count } pliki.
        [many] Pobrano { $count } plików.
       *[other] Pobrano { $count } plików.
    }
```

Runtime results:
- `$count = 1` → "Pobrano 1 plik." (one)
- `$count = 3` → "Pobrano 3 pliki." (few)
- `$count = 5` → "Pobrano 5 plików." (many)
- `$count = 22` → "Pobrano 22 pliki." (few -- ends in 2, not 12)
- `$count = 12` → "Pobrano 12 plików." (many -- ends in 12)

### Arabic (all six categories)

```ftl
# Arabic: uses all six CLDR categories
messages =
    { $count ->
        [zero] لا توجد رسائل.
        [one] رسالة واحدة.
        [two] رسالتان.
        [few] { $count } رسائل.
        [many] { $count } رسالة.
       *[other] { $count } رسالة.
    }
```

---

## Exact Numeric Matches

### Exact match overrides CLDR category

```ftl
items-in-cart =
    { $count ->
        [0] Your cart is empty.
        [1] You have one item in your cart.
       *[other] You have { $count } items in your cart.
    }
```

Resolution order:
1. `$count = 0` → matches `[0]` (exact) BEFORE CLDR `other`
2. `$count = 1` → matches `[1]` (exact) BEFORE CLDR `one`
3. `$count = 5` → no exact match → CLDR resolves to `other`

### Combining exact matches with CLDR categories

```ftl
photos =
    { $count ->
        [0] No photos yet. Upload your first!
        [one] You have one photo.
       *[other] You have { $count } photos.
    }
```

Here `[0]` is an exact match and `[one]` is a CLDR category. The value 1 matches `[one]` via CLDR. The value 0 matches `[0]` via exact match (it would otherwise fall to `*[other]` since 0 is `other` in English).

### Negative number matches

```ftl
temperature =
    { $degrees ->
        [0] Freezing point.
       *[other] { $degrees } degrees.
    }
```

Variant keys support negative numbers: `[-1]` is a valid key for exact matching.

---

## Ordinal Plurals

### English ordinal suffixes

```ftl
your-rank = { NUMBER($pos, type: "ordinal") ->
    [1] You finished first!
    [one] You finished {$pos}st
    [two] You finished {$pos}nd
    [few] You finished {$pos}rd
   *[other] You finished {$pos}th
}
```

Runtime results:
- `$pos = 1` → "You finished first!" (exact match `[1]`)
- `$pos = 2` → "You finished 2nd" (ordinal `two`)
- `$pos = 3` → "You finished 3rd" (ordinal `few`)
- `$pos = 4` → "You finished 4th" (ordinal `other`)
- `$pos = 11` → "You finished 11th" (ordinal `other`, not `one`)
- `$pos = 12` → "You finished 12th" (ordinal `other`, not `two`)
- `$pos = 21` → "You finished 21st" (ordinal `one`)
- `$pos = 22` → "You finished 22nd" (ordinal `two`)
- `$pos = 23` → "You finished 23rd" (ordinal `few`)

### Floor placement

```ftl
floor-label = { NUMBER($floor, type: "ordinal") ->
    [one] {$floor}st floor
    [two] {$floor}nd floor
    [few] {$floor}rd floor
   *[other] {$floor}th floor
}
```

---

## String Selectors

### Gender selection

```ftl
welcome =
    { $gender ->
        [masculine] He has joined the room.
        [feminine] She has joined the room.
       *[other] They have joined the room.
    }
```

The application passes `$gender` as a string: `"masculine"`, `"feminine"`, or any other value. Non-matching strings fall to `*[other]`.

### Status-based selection

```ftl
account-status =
    { $status ->
        [active] Your account is active.
        [suspended] Your account has been suspended. Contact support.
        [trial] Your trial expires in { $daysLeft } days.
       *[other] Unknown account status.
    }
```

### User role selection

```ftl
dashboard-greeting =
    { $role ->
        [admin] Welcome, administrator. You have full access.
        [editor] Welcome, editor. You can manage content.
        [viewer] Welcome. You have read-only access.
       *[other] Welcome to the dashboard.
    }
```

---

## Formatted Number Selectors

### Score with decimal formatting

```ftl
your-score =
    { NUMBER($score, minimumFractionDigits: 1) ->
        [0.0] You scored zero points. What happened?
       *[other] You scored { NUMBER($score, minimumFractionDigits: 1) } points.
    }
```

The NUMBER function formats the value before matching. With `$score = 0`, the formatted value is `"0.0"`, matching `[0.0]`.

### Percentage display

```ftl
completion =
    { NUMBER($percent, maximumFractionDigits: 0) ->
        [0] Not started yet.
        [100] Complete!
       *[other] { NUMBER($percent, maximumFractionDigits: 0) }% complete.
    }
```

---

## Nested Selectors

### Gender + plural (two levels)

```ftl
shared-photos =
    { $gender ->
        [masculine] { $count ->
            [one] He shared one photo.
           *[other] He shared { $count } photos.
        }
        [feminine] { $count ->
            [one] She shared one photo.
           *[other] She shared { $count } photos.
        }
       *[other] { $count ->
            [one] They shared one photo.
           *[other] They shared { $count } photos.
        }
    }
```

Every inner select expression MUST have its own `*[default]` variant. There are three outer variants, each containing a complete inner select expression.

### Platform + count

```ftl
download-prompt =
    { $platform ->
        [windows] { $count ->
            [one] Download one file to your PC?
           *[other] Download { $count } files to your PC?
        }
        [macos] { $count ->
            [one] Download one file to your Mac?
           *[other] Download { $count } files to your Mac?
        }
       *[other] { $count ->
            [one] Download one file?
           *[other] Download { $count } files?
        }
    }
```

---

## Term Attributes as Selectors

### Gender from term attribute

```ftl
-brand-name = Aurora
    .gender = feminine

update-successful =
    { -brand-name.gender ->
        [masculine] { -brand-name } has been updated (m).
        [feminine] { -brand-name } has been updated (f).
       *[other] { -brand-name } has been updated.
    }
```

Term attributes (like `.gender`) are private -- they can ONLY be used as selectors, never displayed directly. The attribute value `"feminine"` is matched against the variant keys.

---

## Official Sources

- https://projectfluent.org/fluent/guide/selectors.html
- https://projectfluent.org/fluent/guide/builtins.html
- https://unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html
