# Term Examples

## Basic Terms

### Brand Name Consistency

```ftl
# Define brand terms once
-brand-name = Firefox
-vendor-name = Mozilla

# Reference in messages — consistent everywhere
about = About { -brand-name }.
update-successful = { -brand-name } has been updated.
rights = { -brand-name } is made by { -vendor-name }.
download-prompt = Download { -brand-name } today.
```

Changing `-brand-name = Firefox` to `-brand-name = Iceweasel` updates ALL referencing messages automatically.

### Multiple Related Terms

```ftl
-app-name = Thunderbird
-company-name = Mozilla Foundation
-support-email = support@example.com
-website-url = https://www.thunderbird.net

welcome = Welcome to { -app-name } by { -company-name }.
help = For help, visit { -website-url } or email { -support-email }.
```

---

## Parameterized Terms

### Grammatical Cases (Polish Example)

```ftl
# Polish: nouns decline by grammatical case
-brand-name =
    { $case ->
       *[nominative] Firefox
        [genitive] Firefoksa
        [dative] Firefoksowi
        [accusative] Firefoksa
        [locative] Firefoksie
        [instrumental] Firefoksem
    }

# Each message passes the required case as a literal string
about = Informacje o { -brand-name(case: "locative") }.
download = Pobierz { -brand-name(case: "accusative") }.
made-by = Stworzone przez { -brand-name(case: "accusative") }.
update-for = Aktualizacja dla { -brand-name(case: "genitive") }.

# No argument → default (*[nominative]) is selected
welcome = { -brand-name } jest gotowy.
```

### Dynamic Value Construction

```ftl
-https = https://{ $host }

visit-main = Visit { -https(host: "example.com") } for more information.
visit-docs = Check { -https(host: "docs.example.com") } for documentation.
visit-api = API reference at { -https(host: "api.example.com") }.
```

### Multiple Parameters

```ftl
-date-format =
    { $style ->
       *[short] { $month }/{ $day }
        [long] { $month } { $day }, { $year }
    }

# All argument values MUST be literals
event-date = Event: { -date-format(style: "long", month: "March", day: "15", year: "2025") }.
```

---

## Term Attributes

### Gender Metadata

```ftl
-brand-name = Aurora
    .gender = feminine

# Use .gender attribute as selector
update-successful =
    { -brand-name.gender ->
        [masculine] { -brand-name } został zaktualizowany.
        [feminine] { -brand-name } została zaktualizowana.
       *[other] Program { -brand-name } został zaktualizowany.
    }

# Same pattern for other messages
crash-report =
    { -brand-name.gender ->
        [masculine] { -brand-name } uległ awarii.
        [feminine] { -brand-name } uległa awarii.
       *[other] Program { -brand-name } uległ awarii.
    }
```

### Multiple Attributes

```ftl
-product-name = Thunderbird
    .gender = masculine
    .animacy = inanimate
    .starts-with-vowel = no

# Select on gender
updated =
    { -product-name.gender ->
        [masculine] { -product-name } został zaktualizowany.
        [feminine] { -product-name } została zaktualizowana.
       *[other] { -product-name } zostało zaktualizowane.
    }

# Select on a different attribute
article =
    { -product-name.starts-with-vowel ->
        [yes] an { -product-name }
       *[no] a { -product-name }
    }
```

### Combined Value and Attributes

```ftl
-os-name = Windows
    .gender = masculine
    .article = no

-os-name-mac = macOS
    .gender = neuter
    .article = no

-os-name-linux = Linux
    .gender = masculine
    .article = yes
```

Each term has its own value (the OS name) plus grammatical metadata as attributes. Other messages select on the attributes to produce grammatically correct sentences.

---

## Term References in Messages

### Simple Reference

```ftl
-brand-name = Firefox

# The term value is interpolated directly
about = About { -brand-name }.
# Result: "About Firefox."
```

### Attribute Reference as Selector

```ftl
-brand-name = Firefox
    .gender = masculine

greeting =
    { -brand-name.gender ->
        [masculine] { -brand-name } est mis à jour.
        [feminine] { -brand-name } est mise à jour.
       *[other] { -brand-name } a été mis à jour.
    }
```

The expression `{ -brand-name.gender }` resolves to the string `"masculine"`, which is then used to select the matching variant.

### Fallback Behavior

When a term is referenced but not defined in the bundle, the Fluent resolver produces the term identifier as fallback text and logs an error. It does NOT throw an exception.

```ftl
# If -missing-term is not defined:
message = Hello { -missing-term }.
# Result: "Hello -missing-term."
# An error is logged in the errors array
```
