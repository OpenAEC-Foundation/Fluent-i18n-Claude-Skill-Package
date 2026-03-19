# Vooronderzoek: Project Fluent (Mozilla i18n)

> **Research date**: 2026-03-19
> **Version**: 1.0 — Combined from 3 research fragments
> **Sources**: projectfluent.org (syntax guide, EBNF spec), GitHub projectfluent/fluent.js (source code, wiki, issues, changelogs), npm registry
> **All claims verified via WebFetch against official sources**

---

## Summary of Key Facts

| Fact | Value |
|------|-------|
| FTL Syntax version | 1.0 (released April 2019) |
| @fluent/bundle version | 0.19.1 (released April 2, 2025) |
| @fluent/bundle license | Apache-2.0 OR MIT |
| @fluent/bundle Node.js req | ^20.19 \|\| ^22.12 \|\| >=24 |
| @fluent/syntax version | 0.19.0 |
| @fluent/react version | 0.15.2 |
| @fluent/react license | Apache-2.0 |
| @fluent/react React peer dep | >=16.8.0 |
| @fluent/react bundle peer dep | >=0.16.0 |
| @fluent/langneg version | 0.7.0 |
| @fluent/langneg license | Apache-2.0 |
| @fluent/langneg dependencies | None (zero runtime dependencies) |
| Package rename | `fluent` → `@fluent/bundle` at v0.13.0 (July 2019) |
| Major API break | v0.14.0 — `addMessages`/`format` removed, two-step API introduced |
| TypeScript migration | v0.15.0 (January 2020) |
| Temporal support | v0.19.0+ (FluentDateTime accepts TC39 Temporal objects) |
| Primary React components | `LocalizationProvider`, `Localized` |
| Primary React hook | `useLocalization` → `{ l10n }` |
| Primary React class | `ReactLocalization(bundles, parseMarkup?, reportError?)` |
| Negotiation strategies | filtering (default), matching, lookup |
| SSR requirements | Custom `parseMarkup`, `acceptedLanguages()` |
| Known gaps | No async fallback loading, no RTL support, no ref forwarding in HOC |
| Critical API signatures | `FluentBundle(locales, options?)`, `addResource(res, options?)`, `getMessage(id)`, `formatPattern(pattern, args?, errors?)` |
| FluentFunction signature | `(positional: FluentValue[], named: Record<string, FluentValue>) => FluentValue` |
| negotiateLanguages signature | `(requested, available, options?) => string[]` |
| getString signature | `(id, vars?, fallback?) => string` |

---

<!-- §1 START — FTL Specification and Syntax -->

## §1 FTL Specification and Syntax

> **Source version**: Fluent Syntax 1.0 (released April 2019)
> **Sources fetched**: 2026-03-19
> **All claims sourced from**: projectfluent.org (syntax guide), github.com/projectfluent/fluent (EBNF spec)

---

### §1.1 Design Philosophy

Project Fluent is a localization system created by Mozilla, designed for "natural-sounding translations." Its core mission statement: **"Translators should be able to use the entire expressive power of their language without asking developers for permission."**

Source: https://projectfluent.org

#### Asymmetric Localization

The defining principle of Fluent is **asymmetric localization**. Traditional i18n systems (gettext, Java properties, iOS .strings) enforce a one-to-one mapping between source strings and translations. Fluent breaks this constraint:

- A simple string in English can map to a complex multi-variant translation in another language.
- Natural-sounding translations with genders and grammatical cases are used only when necessary.
- Expressiveness is not limited by the grammar of the source language.
- Locale-specific logic is isolated so it does not leak to other locales.

As stated on projectfluent.org: "Software localization has been dominated by an outdated paradigm: translations that map one-to-one to the English copy. The grammar of the source language imposes limits on the expressiveness of the translation." Fluent eliminates this limitation.

#### Key Design Principles

1. **Progressive Enhancement**: Each language develops independently without impacting others. English may use a simple string while Polish uses a complex selector with grammatical cases.
2. **Simplicity for simple cases**: The basic `key = value` syntax is as simple as any properties file. Complexity is added only when needed.
3. **Fully-featured**: Date, time, and number formatting. Plural categories. Bidirectionality support. Custom formatters. Robust error handling.
4. **Terms system**: Reusable message fragments that enforce consistency across translations (brand names, product terms).

Source: https://projectfluent.org

---

### §1.2 Message Syntax

The basic unit of translation in Fluent is called a **message**. Messages are containers for information, consisting of an identifier, an equals sign, and a value (called a **pattern**).

Source: https://projectfluent.org/fluent/guide/hello.html

#### Simple Messages

```ftl
hello = Hello, world!
```

The text value begins at the first non-blank character after the `=` sign. The identifier is what developers use to reference the message in code.

#### Multiline Messages

Text can span multiple lines as long as each continuation line is indented by at least one space. ONLY space characters (U+0020) count as indentation; tabs are treated as regular text characters.

Source: https://projectfluent.org/fluent/guide/multiline.html

```ftl
single = Text can be written in a single line.

multi = Text can also span multiple lines
    as long as each new line is indented
    by at least one space.

block =
    Sometimes it's more readable to format
    multiline text as a "block", which means
    starting it on a new line. All lines must
    be indented by at least one space.
```

#### Whitespace and Dedentation in Multiline

Patterns start and end at the first and last non-blank characters respectively. Leading and trailing blanks are ignored. Line breaks and blank lines are preserved ONLY when positioned inside multiline text (text before AND after them).

```ftl
leading-spaces =     This message's value starts with the word "This".

leading-lines =


    This message's value starts with the word "This".
    The blank lines under the identifier are ignored.

blank-lines =

    The blank line above this line is ignored.
    This is a second line of the value.

    The blank line above this line is preserved.
```

**Dedentation rule**: In multiline patterns, all common indent is removed. The first line (starting on the same line as the identifier) is excluded from dedentation calculations.

```ftl
multiline1 =
    This message has 4 spaces of indent
        on the second line of its value.
```

In the above example, the common indent of all indented lines is 4 spaces, so 4 spaces are removed from every line. The second line retains 4 extra spaces of visible indent.

Source: https://projectfluent.org/fluent/guide/multiline.html

#### Message References

One message can be embedded inside another using placeable syntax:

```ftl
menu-save = Save
help-menu-save = Click { menu-save } to save the file.
```

This keeps translations consistent and makes maintenance easier.

Source: https://projectfluent.org/fluent/guide/references.html

#### Formal Grammar (Messages)

From the EBNF specification:

```ebnf
Message ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*
```

A message MUST start with a letter. It may contain letters, digits, underscores, and hyphens. A message MUST have either a value (Pattern), attributes, or both.

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

### §1.3 Placeables and Variables

Placeables are syntax elements denoted by curly braces `{` and `}` that incorporate dynamic content into text. They enable variable interpolation, message/term references, and function calls.

Source: https://projectfluent.org/fluent/guide/placeables.html

#### Variable References

Variables are external data provided by the application at runtime. They use the `$variable-name` syntax:

```ftl
welcome = Welcome, { $user }!
unread-emails = { $user } has { $email-count } unread emails.
```

When rendered with `$user = "Jane"` and `$email-count = 5`:
```
Welcome, Jane!
Jane has 5 unread emails.
```

Variables may dynamically change as the user interacts with the product. They can represent user names, counts, battery levels, timestamps, and other runtime data.

Source: https://projectfluent.org/fluent/guide/variables.html

#### Automatic Number and Date Formatting

Numbers and dates are automatically formatted according to the user's locale:

```ftl
time-elapsed = Time elapsed: { $duration }s.
```

With `$duration = 12345.678`:
- American English: `Time elapsed: 12,345.678s.`
- South African English: `Time elapsed: 12 345,678s.`

Source: https://projectfluent.org/fluent/guide/variables.html

#### Explicit Formatting Control

Built-in functions allow overriding automatic formatting:

```ftl
time-elapsed = Time elapsed: { NUMBER($duration, maximumFractionDigits: 0) }s.
```

This produces `Time elapsed: 12,345s.` regardless of the default fraction digit count.

#### Formal Grammar (Placeables and Variables)

```ebnf
inline_placeable  ::= "{" blank? (SelectExpression | InlineExpression) blank? "}"
VariableReference ::= "$" Identifier
InlineExpression  ::= StringLiteral | NumberLiteral | FunctionReference
                    | MessageReference | TermReference | VariableReference
                    | inline_placeable
```

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

### §1.4 Select Expressions and Plurals

Selectors allow defining multiple translation variants based on a runtime value. One variant MUST be marked as the default with the `*` prefix.

Source: https://projectfluent.org/fluent/guide/selectors.html

#### Basic Select Expression

```ftl
emails =
    { $unreadEmails ->
        [one] You have one unread email.
       *[other] You have { $unreadEmails } unread emails.
    }
```

The selector expression (`$unreadEmails`) is evaluated, and the matching variant key is selected. If no key matches, the default variant (marked with `*`) is used.

#### Selector Types

1. **String selectors**: Match directly against variant keys as string values.
2. **Numeric selectors**: Match either exact number values OR CLDR plural categories.

The CLDR plural categories are: `zero`, `one`, `two`, `few`, `many`, `other`. Which categories a language uses depends on its plural rules. English uses only `one` and `other`.

#### Formatted Number Selectors

When numbers need specific formatting in the selector, apply formatting options directly:

```ftl
your-score =
    { NUMBER($score, minimumFractionDigits: 1) ->
        [0.0]   You scored zero points. What happened?
       *[other] You scored { NUMBER($score, minimumFractionDigits: 1) } points.
    }
```

The formatted number determines which CLDR category applies. This may differ from the unformatted value's category.

#### Ordinal Plurals

Ordinal selectors use `type: "ordinal"` to access ordinal plural categories:

```ftl
your-rank = { NUMBER($pos, type: "ordinal") ->
   [1] You finished first!
   [one] You finished {$pos}st
   [two] You finished {$pos}nd
   [few] You finished {$pos}rd
  *[other] You finished {$pos}th
}
```

Note: Exact numeric matches (like `[1]`) are checked BEFORE CLDR category matches. This allows special-casing specific values while falling back to category-based pluralization.

#### Formal Grammar (Selectors)

```ebnf
SelectExpression ::= InlineExpression blank? "->" blank_inline? variant_list
variant_list     ::= Variant* DefaultVariant Variant* line_end
Variant          ::= line_end blank? VariantKey blank_inline? Pattern
DefaultVariant   ::= line_end blank? "*" VariantKey blank_inline? Pattern
VariantKey       ::= "[" blank? (NumberLiteral | Identifier) blank? "]"
```

Key observations from the grammar:
- `variant_list` requires exactly ONE `DefaultVariant` (with `*` prefix).
- Variant keys can be either `NumberLiteral` (for exact matches) or `Identifier` (for CLDR categories or string matching).
- The default variant can appear at any position in the list, not necessarily last.

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

### §1.5 Terms and References

Terms are specialized constructs that begin with a hyphen (`-`) and can ONLY be used as references in other messages. They cannot be retrieved directly by the localization runtime.

Source: https://projectfluent.org/fluent/guide/terms.html

#### Basic Term Syntax

```ftl
-brand-name = Firefox

about = About { -brand-name }.
update-successful = { -brand-name } has been updated.
```

#### Key Differences from Messages

| Aspect | Messages | Terms |
|--------|----------|-------|
| Prefix | None (starts with letter) | `-` (hyphen) |
| Runtime access | Can be retrieved directly | Cannot be retrieved directly |
| Purpose | User-facing text | Shared vocabulary / brand consistency |
| Variable data | Received from application | Received from referencing messages |

#### Parameterized Terms

Terms accept parameters passed from referencing messages. This enables grammatical case handling:

```ftl
-brand-name =
    { $case ->
       *[nominative] Firefox
        [locative] Firefoksie
    }

about = Informacje o { -brand-name(case: "locative") }.
```

Terms can also accept parameters for constructing dynamic values:

```ftl
-https = https://{ $host }
visit = Visit { -https(host: "example.com") } for more information.
```

#### Term Attributes for Grammatical Metadata

Terms can store grammatical metadata (gender, animacy, etc.) as attributes. These attributes are private and can ONLY be used as selectors — they are NOT accessible to the localization runtime.

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

#### Formal Grammar (Terms)

```ebnf
Term          ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
TermReference ::= "-" Identifier AttributeAccessor? CallArguments?
CallArguments ::= blank? "(" blank? argument_list blank? ")"
NamedArgument ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
```

Key observations:
- A Term MUST have a value (Pattern). Unlike Messages, a Term cannot be attribute-only.
- `TermReference` can have BOTH an `AttributeAccessor` AND `CallArguments`, but in practice accessing an attribute AND passing arguments simultaneously is unusual.
- `CallArguments` for terms ONLY accept `NamedArgument` (not positional), with values restricted to `StringLiteral` or `NumberLiteral`.

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

### §1.6 Attributes

Attributes allow multiple translatable strings to be associated with a single message or term, using the `.attribute` syntax on indented continuation lines.

Source: https://projectfluent.org/fluent/guide/attributes.html

#### Attribute Syntax

```ftl
login-input = Predefined value
    .placeholder = email@example.com
    .aria-label = Login input value
    .title = Type your login email
```

This groups four related strings under one message identifier. Localization tools can display and manage them together.

#### Use Cases

UI elements often contain multiple translatable strings per widget. An HTML form input may have a value, a `placeholder` attribute, an `aria-label` attribute, and a `title` attribute. Grouping them under one message improves maintainability.

#### Attributes on Terms

Terms can also have attributes. Term attributes are **private** — they cannot be retrieved by the localization runtime and can ONLY be used as selectors (see section §1.5 for the gender example).

#### Value-less Messages with Attributes

From the formal grammar, a Message can have ONLY attributes and no value:

```ebnf
Message ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
```

This means a message like the following is valid:

```ftl
login-input =
    .placeholder = email@example.com
    .aria-label = Login input value
```

Here `login-input` has no value, only attributes. This is useful when a UI element has no visible text content but does have accessible labels.

#### Formal Grammar (Attributes)

```ebnf
Attribute         ::= line_end blank? "." Identifier blank_inline? "=" blank_inline? Pattern
AttributeAccessor ::= "." Identifier
```

An attribute starts on a new line, is indented, and begins with a dot (`.`) followed by an identifier.

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

### §1.7 Comments

Comments in Fluent use the `#` character and come in three levels, each serving a different organizational purpose.

Source: https://projectfluent.org/fluent/guide/comments.html

#### Comment Levels

| Syntax | Name | Scope | Binding |
|--------|------|-------|---------|
| `#` | Comment | Message-level | Bound to the next message if directly above it |
| `##` | Group Comment | Section-level | Always standalone; divides file into groups |
| `###` | Resource Comment | File-level | Always standalone; describes entire file |

#### Complete Example

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

#### Comment Binding Rules

- A single-hash comment (`#`) placed **directly above** a message (no blank line between them) is considered part of that message. Localization tools present the comment together with the message.
- If there IS a blank line between a `#` comment and a message, the comment is standalone.
- `##` and `###` comments are ALWAYS standalone and never bind to messages.

#### Formal Grammar (Comments)

```ebnf
CommentLine ::= ("###" | "##" | "#") ("\u0020" comment_char*)? line_end
comment_char ::= any_char - line_end
```

Note: The space after `#` / `##` / `###` is significant. A comment line is either the hash characters alone or the hash characters followed by a single space and then comment text. This means `#comment` (no space) is NOT valid — it must be `# comment`.

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

### §1.8 Built-in Functions (NUMBER, DATETIME)

Fluent provides two built-in functions for explicit formatting of numeric and date/time values. In most cases, Fluent automatically selects the right formatter for the argument and formats it appropriately for the user's locale. The built-in functions give translators control when automatic formatting is insufficient.

Source: https://projectfluent.org/fluent/guide/builtins.html

#### NUMBER()

Formats a number to a string in a given locale. Accepts the following named arguments (based on Intl.NumberFormat options):

```ftl
emails = You have { NUMBER($unreadEmails) } unread emails.

score = Your score: { NUMBER($score, minimumFractionDigits: 1, maximumFractionDigits: 1) }.
```

Known arguments:
- `minimumFractionDigits` — Minimum number of fraction digits
- `maximumFractionDigits` — Maximum number of fraction digits
- `type` — Can be set to `"ordinal"` for ordinal plural matching (see section §1.4)
- `minimumIntegerDigits` — Minimum number of integer digits
- `useGrouping` — Whether to use grouping separators (thousands)

The NUMBER function can be used in two contexts:
1. **In placeables** — for display formatting: `{ NUMBER($count, minimumFractionDigits: 2) }`
2. **In selectors** — for plural category determination: `{ NUMBER($count, type: "ordinal") -> ... }`

#### DATETIME()

Formats a date to a string in a given locale. Accepts arguments matching Intl.DateTimeFormat options:

```ftl
last-notice = Last checked: { DATETIME($lastChecked, day: "numeric", month: "long") }.
```

Known arguments:
- `day` — `"numeric"`, `"2-digit"`
- `month` — `"numeric"`, `"2-digit"`, `"narrow"`, `"short"`, `"long"`
- `year` — `"numeric"`, `"2-digit"`
- `hour` — `"numeric"`, `"2-digit"`
- `minute` — `"numeric"`, `"2-digit"`
- `second` — `"numeric"`, `"2-digit"`
- `weekday` — `"narrow"`, `"short"`, `"long"`
- `era` — `"narrow"`, `"short"`, `"long"`
- `timeZoneName` — `"short"`, `"long"`

#### Formal Grammar (Functions)

```ebnf
FunctionReference ::= Identifier CallArguments
CallArguments     ::= blank? "(" blank? argument_list blank? ")"
argument_list     ::= (Argument blank? "," blank?)* Argument?
Argument          ::= NamedArgument | InlineExpression
NamedArgument     ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
```

Note: Function names in the grammar are plain `Identifier` values (uppercase by convention: `NUMBER`, `DATETIME`). The grammar does not enforce uppercase, but it is the universal convention.

Source: https://projectfluent.org/fluent/guide/builtins.html, https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

### §1.9 Custom Functions

The Fluent specification explicitly supports extensibility. From the documentation: "The list of available functions is extensible and environments may want to introduce additional functions, designed to aid localizers writing translations targeted for such environments."

Source: https://projectfluent.org/fluent/guide/functions.html

#### What the Specification Defines

The FTL syntax itself does NOT define how custom functions are registered — that is implementation-specific. What FTL defines is:

1. Functions are called with `FUNCTION_NAME(arguments)` syntax inside placeables.
2. Arguments can be positional (`InlineExpression`) or named (`NamedArgument`).
3. Named argument values MUST be string literals or number literals (not variable references).

#### Registration in @fluent/bundle (JavaScript)

Custom functions are registered when creating a `FluentBundle` instance. The function receives positional arguments and a named arguments object. This is covered in detail in the @fluent/bundle API documentation, not in the FTL syntax guide itself.

**Documentation gap**: The official syntax guide at projectfluent.org does NOT provide detailed guidance on implementing custom functions. It only states the extensibility principle. Implementation details must be sourced from the individual runtime libraries (@fluent/bundle for JS, fluent-rs for Rust, etc.).

---

### §1.10 Whitespace and Indentation Rules

Whitespace handling in Fluent is precise and has several rules that are critical to understand.

Source: https://projectfluent.org/fluent/guide/multiline.html, EBNF spec

#### Core Rules

1. **Only spaces (U+0020) count as indentation.** Tabs are NOT indentation — they are treated as regular text characters.
2. **Continuation lines MUST be indented by at least one space** to be part of the preceding pattern.
3. **Common indent is stripped (dedented)** from all indented lines in a multiline pattern.
4. **The first line** (on the same line as `=`) is NOT considered indented and is excluded from dedentation calculations.
5. **Leading blank lines** (between `=` and the first content line) are ignored.
6. **Trailing blank lines** after the last content line are ignored.
7. **Interior blank lines** (with content both before and after) are preserved.

#### Formal Whitespace Productions

```ebnf
blank_inline ::= "\u0020"+
line_end     ::= "\u000D\u000A" | "\u000A" | EOF
blank_block  ::= (blank_inline? line_end)+
blank        ::= (blank_inline | line_end)+
```

#### Preserving Leading Whitespace

To force leading spaces in output, use quoted text:

```ftl
blank-is-preserved = {"    "}This message starts with 4 spaces.
```

Without the quoted text placeable, the leading spaces would be stripped by the dedentation algorithm.

#### Block Text vs. Inline Text

From the EBNF:

```ebnf
PatternElement ::= inline_text | block_text | inline_placeable | block_placeable
inline_text    ::= text_char+
block_text     ::= blank_block blank_inline indented_char inline_text?
indented_char  ::= text_char - "[" - "*" - "."
```

The `indented_char` production is notable: the first character of a continuation line CANNOT be `[`, `*`, or `.`. These characters would be interpreted as variant keys, default variant markers, or attribute accessors respectively. To start a line with these characters, use a quoted text placeable:

```ftl
leading-bracket =
    This message has an opening square bracket
    at the beginning of the third line:
    {"["} like this.
```

---

### §1.11 Key Differences from ICU MessageFormat

Based on the design philosophy and syntax analysis, here are the key differences between Fluent and ICU MessageFormat:

| Aspect | ICU MessageFormat | Project Fluent |
|--------|-------------------|----------------|
| **Localization model** | Symmetric (1:1 mapping) | Asymmetric (1:N mapping) |
| **Complexity leakage** | Source language grammar leaks to all translations | Locale logic is isolated per language |
| **File format** | Embedded in code or properties files | Dedicated `.ftl` files |
| **Terms/brands** | No built-in concept | First-class `-term` syntax |
| **Attributes** | Not supported | `.attribute` syntax for multi-value messages |
| **Comments** | Not standardized | Three-level comment system (#, ##, ###) |
| **Error handling** | Typically throws errors | Graceful fallback (shows message ID) |
| **Plural syntax** | `{count, plural, one{...} other{...}}` | `{ $count -> [one] ... *[other] ... }` |
| **Gender syntax** | `{gender, select, male{...} female{...} other{...}}` | Uses term attributes with selectors |
| **Nesting** | Deep nesting common and hard to read | Flat structure, complexity in variants |
| **Translator access** | Often requires developer coordination | Translators can add complexity independently |

The most significant architectural difference is that in ICU MessageFormat, if Polish needs grammatical cases, the developer must modify the source code to pass the case parameter and update the English source string to include a `select` statement (even if English doesn't use cases). In Fluent, the Polish translator adds a selector independently — the English file stays simple, and no code changes are needed.

Source: https://projectfluent.org (design philosophy), syntax guide analysis

---

### §1.12 Edge Cases and Gotchas

#### 1. Special Characters at Line Start

The first character of a continuation line (after indentation) CANNOT be `[`, `*`, or `.` — these are reserved for variant keys, default variant markers, and attribute accessors. Use quoted text placeables to escape them:

```ftl
# WRONG - "[" will be parsed as variant key start
bracket-wrong =
    This is the first line
    [This would break]

# CORRECT
bracket-correct =
    This is the first line
    {"["}This works fine]
```

Source: EBNF `indented_char ::= text_char - "[" - "*" - "."`

#### 2. Curly Braces in Text

Literal `{` and `}` characters MUST be escaped using quoted text placeables:

```ftl
opening-brace = This message features an opening curly brace: {"{"}.
closing-brace = This message features a closing curly brace: {"}"}.
```

Source: https://projectfluent.org/fluent/guide/special.html, EBNF `special_text_char ::= "{" | "}"`

#### 3. Escape Sequences (Only in Quoted Text)

Escape sequences work ONLY inside quoted text (string literals), NOT in regular text:

| Sequence | Meaning |
|----------|---------|
| `\"` | Literal double quote |
| `\\` | Literal backslash |
| `\uHHHH` | Unicode code point U+0000 to U+FFFF |
| `\UHHHHHH` | Any Unicode code point (6 hex digits) |

```ftl
privacy-label = Privacy{"\u00A0"}Policy
```

In regular (unquoted) text, the backslash `\` is just a regular character with no special meaning.

Source: https://projectfluent.org/fluent/guide/special.html

#### 4. Junk Handling

If the parser encounters syntax it cannot understand, it produces `Junk` entries rather than failing entirely:

```ebnf
Junk      ::= junk_line (junk_line - "#" - "-" - [a-zA-Z])*
junk_line ::= /[^\n]*/ ("\u000A" | EOF)
```

Junk continues until the parser finds a line starting with `#`, `-`, or a letter (which could be a new entry). This means a syntax error in one message does NOT break parsing of subsequent messages — a critical robustness feature.

Source: EBNF spec

#### 5. Identifier Restrictions

Message identifiers MUST start with a letter `[a-zA-Z]` and can only contain letters, digits, underscores, and hyphens. NO dots, spaces, or other special characters:

```ebnf
Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*
```

Term identifiers follow the same rule but are prefixed with `-` at the syntax level (the `-` is NOT part of the Identifier production).

#### 6. Named Argument Values Are Restricted

In function calls and term references, named argument values MUST be string literals or number literals:

```ebnf
NamedArgument ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
```

You CANNOT pass a variable reference as a named argument value. This is NOT valid: `{ -term(case: $myCase) }`. The value must be a literal like `{ -term(case: "nominative") }`.

#### 7. Default Variant is Mandatory

Every select expression MUST have exactly one default variant (marked with `*`). The grammar enforces this:

```ebnf
variant_list ::= Variant* DefaultVariant Variant* line_end
```

Omitting the default variant is a syntax error.

#### 8. Tab Characters

Tabs (U+0009) are NOT whitespace for indentation purposes. A tab at the start of a continuation line will be treated as text content, not as indent. ALWAYS use spaces (U+0020) for indentation.

#### 9. Comment Space Requirement

The comment syntax requires a space after the hash characters:

```ebnf
CommentLine ::= ("###" | "##" | "#") ("\u0020" comment_char*)? line_end
```

`#comment` is NOT valid. It must be `# comment` (with a space). An empty comment (`#` alone on a line) IS valid.

#### 10. Message vs. Term Value Requirements

A Message can exist with ONLY attributes and no value. A Term MUST have a value:

```ebnf
Message ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
Term    ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
```

This is because terms are designed to be referenced in placeables, which requires a value to interpolate.

---

### §1.13 Appendix: Complete EBNF Grammar (Fluent Syntax 1.0)

For reference, the complete formal grammar as published in the specification repository:

```ebnf
Resource            ::= (Entry | blank_block | Junk)*
Entry               ::= (Message line_end)
                      | (Term line_end)
                      | CommentLine
Message             ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
Term                ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
CommentLine         ::= ("###" | "##" | "#") ("\u0020" comment_char*)? line_end
comment_char        ::= any_char - line_end
Junk                ::= junk_line (junk_line - "#" - "-" - [a-zA-Z])*
junk_line           ::= /[^\n]*/ ("\u000A" | EOF)
Attribute           ::= line_end blank? "." Identifier blank_inline? "=" blank_inline? Pattern
Pattern             ::= PatternElement+
PatternElement      ::= inline_text
                      | block_text
                      | inline_placeable
                      | block_placeable
inline_text         ::= text_char+
block_text          ::= blank_block blank_inline indented_char inline_text?
inline_placeable    ::= "{" blank? (SelectExpression | InlineExpression) blank? "}"
block_placeable     ::= blank_block blank_inline? inline_placeable
InlineExpression    ::= StringLiteral
                      | NumberLiteral
                      | FunctionReference
                      | MessageReference
                      | TermReference
                      | VariableReference
                      | inline_placeable
StringLiteral       ::= "\"" quoted_char* "\""
NumberLiteral       ::= "-"? digits ("." digits)?
FunctionReference   ::= Identifier CallArguments
MessageReference    ::= Identifier AttributeAccessor?
TermReference       ::= "-" Identifier AttributeAccessor? CallArguments?
VariableReference   ::= "$" Identifier
AttributeAccessor   ::= "." Identifier
CallArguments       ::= blank? "(" blank? argument_list blank? ")"
argument_list       ::= (Argument blank? "," blank?)* Argument?
Argument            ::= NamedArgument
                      | InlineExpression
NamedArgument       ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
SelectExpression    ::= InlineExpression blank? "->" blank_inline? variant_list
variant_list        ::= Variant* DefaultVariant Variant* line_end
Variant             ::= line_end blank? VariantKey blank_inline? Pattern
DefaultVariant      ::= line_end blank? "*" VariantKey blank_inline? Pattern
VariantKey          ::= "[" blank? (NumberLiteral | Identifier) blank? "]"
Identifier          ::= [a-zA-Z] [a-zA-Z0-9_-]*
digits              ::= [0-9]+
blank_inline        ::= "\u0020"+
line_end            ::= "\u000D\u000A" | "\u000A" | EOF
blank_block         ::= (blank_inline? line_end)+
blank               ::= (blank_inline | line_end)+
```

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

<!-- §2 START — @fluent/bundle API -->

## §2 @fluent/bundle API

> **Research date**: 2026-03-19
> **Sources**: All claims verified via WebFetch against official GitHub sources and npm registry.

---

### §2.1 Package Overview and Version

**Current version**: 0.19.1 (released April 2, 2025)
**npm package**: `@fluent/bundle`
**Repository**: https://github.com/projectfluent/fluent.js/tree/main/fluent-bundle
**License**: Apache-2.0 OR MIT
**Node.js requirement**: `^20.19 || ^22.12 || >=24`

The `@fluent/bundle` package is the core JavaScript runtime for Project Fluent. It provides the `FluentBundle` class that parses FTL resources and formats localized messages at runtime. The package was renamed from `fluent` to `@fluent/bundle` in version 0.13.0 (July 2019).

**Entry points** (from package.json):
- `main`: `./index.js` (CommonJS)
- `module`: `./esm/index.js` (ES modules)
- `types`: `./esm/index.d.ts` (TypeScript declarations)

**Runtime requirements**: The library requires the following `Intl` APIs to be available:
- `Intl.DateTimeFormat`
- `Intl.NumberFormat`
- `Intl.PluralRules`

**Public exports** (from `@fluent/bundle`):
- Classes: `FluentBundle`, `FluentResource`, `FluentType`, `FluentNone`, `FluentNumber`, `FluentDateTime`
- Types: `FluentValue`, `FluentVariable`, `FluentFunction`, `TextTransform`, `Message`, `Scope`

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/index.ts

---

### §2.2 FluentBundle Class

The `FluentBundle` class is the central API for formatting Fluent messages. It holds parsed resources, resolves message references, and formats patterns with arguments.

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/bundle.ts

#### §2.2.1 Constructor and Options

```typescript
constructor(
  locales: string | Array<string>,
  options?: {
    functions?: Record<string, FluentFunction>;
    useIsolating?: boolean;
    transform?: TextTransform;
  }
)
```

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `locales` | `string \| Array<string>` | (required) | BCP 47 locale identifier(s) used by `Intl` formatters |
| `functions` | `Record<string, FluentFunction>` | `{}` | Custom functions available in FTL, merged with built-in NUMBER and DATETIME |
| `useIsolating` | `boolean` | `true` | Wrap placeables in Unicode isolation marks (U+2068/U+2069) for bidirectional text |
| `transform` | `TextTransform` | `(v) => v` | Transform applied to all TextElement strings in patterns |

Where `TextTransform` is:
```typescript
type TextTransform = (text: string) => string;
```

**Example — basic bundle creation**:
```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
```

**Example — bundle with custom functions and options**:
```typescript
import { FluentBundle, FluentResource, FluentVariable } from "@fluent/bundle";

const bundle = new FluentBundle("en-US", {
  useIsolating: false,
  functions: {
    UPCASE: (positional, named) => {
      const val = positional[0];
      return typeof val === "string" ? val.toUpperCase() : String(val).toUpperCase();
    },
  },
  transform: (text) => text, // identity transform (default)
});
```

**Internal properties** (public but prefixed with underscore — not part of stable API):
- `locales: Array<string>` — the resolved locale array
- `_terms: Map<string, Term>` — parsed terms (prefixed with `-` in FTL)
- `_messages: Map<string, Message>` — parsed messages
- `_functions: Record<string, FluentFunction>` — merged custom + built-in functions
- `_useIsolating: boolean`
- `_transform: TextTransform`
- `_intls: IntlCache` — memoized Intl formatter instances

#### §2.2.2 addResource()

```typescript
addResource(
  res: FluentResource,
  options?: { allowOverrides?: boolean }
): Array<Error>
```

Adds a `FluentResource` to the bundle. Messages and terms from the resource become available for formatting.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `res` | `FluentResource` | (required) | A parsed FTL resource |
| `allowOverrides` | `boolean` | `false` | If `true`, new messages/terms overwrite existing ones with the same ID. If `false`, duplicates produce an error. |

**Return value**: An array of `Error` objects for any messages that had syntax errors during parsing. Syntax errors are **per-message** — they do not prevent other valid messages in the same resource from being added.

**Example**:
```typescript
const resource = new FluentResource(`
hello = Hello, world!
goodbye = Goodbye, {$name}!
`);

const errors = bundle.addResource(resource);
if (errors.length > 0) {
  console.error("FTL syntax errors:", errors);
}
```

**Example — allowing overrides**:
```typescript
const base = new FluentResource(`hello = Hello`);
const override = new FluentResource(`hello = Hi there`);

bundle.addResource(base);
bundle.addResource(override, { allowOverrides: true });
// "hello" now resolves to "Hi there"
```

#### §2.2.3 getMessage()

```typescript
getMessage(id: string): Message | undefined
```

Retrieves a parsed message by its identifier. Returns `undefined` if the message does not exist. Terms (identifiers starting with `-`) are NOT accessible via `getMessage()`.

The returned `Message` object has this shape:
```typescript
interface Message {
  value: Pattern | null;
  attributes: Record<string, Pattern>;
}
```

- `value` — the main pattern of the message, or `null` if the message only has attributes
- `attributes` — a record of attribute patterns keyed by attribute name

**Example**:
```typescript
const resource = new FluentResource(`
login-input =
    .placeholder = Email address
    .aria-label = Login input
`);
bundle.addResource(resource);

const msg = bundle.getMessage("login-input");
// msg.value is null (message has no value, only attributes)
// msg.attributes["placeholder"] is a Pattern
// msg.attributes["aria-label"] is a Pattern

if (msg) {
  if (msg.value) {
    const text = bundle.formatPattern(msg.value);
  }
  const placeholder = bundle.formatPattern(msg.attributes["placeholder"]);
  // → "Email address"
}
```

#### §2.2.4 hasMessage()

```typescript
hasMessage(id: string): boolean
```

Returns `true` if the bundle contains a message with the given identifier. Like `getMessage()`, this does NOT check for terms.

**Example**:
```typescript
if (bundle.hasMessage("welcome")) {
  const msg = bundle.getMessage("welcome")!;
  const text = bundle.formatPattern(msg.value!);
}
```

#### §2.2.5 formatPattern()

```typescript
formatPattern(
  pattern: Pattern,
  args?: Record<string, FluentVariable> | null,
  errors?: Array<Error> | null
): string
```

The core formatting method. Takes a `Pattern` (obtained from `getMessage()`) and returns a fully resolved string.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `pattern` | `Pattern` | (required) | A pattern from `getMessage().value` or `getMessage().attributes[name]` |
| `args` | `Record<string, FluentVariable> \| null` | `null` | Variables to substitute into the pattern |
| `errors` | `Array<Error> \| null` | `null` | If provided, resolution errors are appended here instead of being thrown |

Where `FluentVariable` is:
```typescript
type FluentVariable = FluentValue | TemporalObject | string | number | Date;
```

And `FluentValue` is:
```typescript
type FluentValue = FluentType<unknown> | string;
```

**Critical behavior regarding errors**: If the `errors` array is NOT provided and a resolution error occurs, `formatPattern` **throws** on the first error. If `errors` IS provided, errors are collected and the formatter returns a best-effort string with `{???}` placeholders for failed parts.

**Example — basic formatting**:
```typescript
const resource = new FluentResource(`
welcome = Welcome, {$name}, to {-brand-name}!
-brand-name = Foo 3000
`);
bundle.addResource(resource);

const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "Anna" });
  // → "Welcome, Anna, to Foo 3000!"
}
```

**Example — with error collection**:
```typescript
const errors: Error[] = [];
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "Anna" }, errors);
  if (errors.length > 0) {
    console.warn("Formatting issues:", errors);
  }
}
```

**Example — formatting with numbers and dates**:
```typescript
const resource = new FluentResource(`
items-count = You have {$count} items.
last-login = Last login: {$date}
price = Total: {NUMBER($amount, minimumFractionDigits: 2)}
`);
bundle.addResource(resource);

const msg = bundle.getMessage("items-count");
if (msg?.value) {
  bundle.formatPattern(msg.value, { count: 5 });
  // → "You have 5 items."
}

const loginMsg = bundle.getMessage("last-login");
if (loginMsg?.value) {
  bundle.formatPattern(loginMsg.value, { date: new Date() });
  // → "Last login: 3/19/2026" (locale-dependent formatting)
}
```

---

### §2.3 FluentResource Class

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/resource.ts

```typescript
class FluentResource {
  body: Array<Message | Term>;
  constructor(source: string);
}
```

`FluentResource` parses an FTL string into an internal representation optimized for runtime use. It uses a purpose-built parser that is smaller and faster than the full `@fluent/syntax` parser but does NOT produce a full AST.

**Key characteristics**:
- The `body` property contains an array of parsed `Message` and `Term` objects
- Parsing errors are **per-message**: a syntax error in one message does not prevent other messages from being parsed
- The parser "minimizes false negatives at the expense of increasing false positives" — it accepts valid messages with near-perfect accuracy but may parse some technically invalid structures
- For strict validation, use `@fluent/syntax` instead

**Example**:
```typescript
import { FluentResource } from "@fluent/bundle";

const resource = new FluentResource(`
hello = Hello, world!
welcome = Welcome, {$name}!
-brand = Acme Corp
`);

console.log(resource.body.length); // 3 (2 messages + 1 term)
```

**Version history note**: Before v0.14.0, `FluentResource` extended `Map` and had a static `fromString()` method. Both were removed. ALWAYS use the constructor directly.

---

### §2.4 Error Handling

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/resolver.ts

Fluent's error handling philosophy is **resilience over failure**. The resolver attempts to salvage as much of the translation as possible, replacing failed parts with `{???}` fallback values.

#### Error Types During Resolution

| Error Type | Condition | Fallback |
|-----------|-----------|----------|
| `ReferenceError` | Unknown variable: `$name` | `{$name}` |
| `ReferenceError` | Unknown message: `msg-id` | `{msg-id}` |
| `ReferenceError` | Unknown term: `-term-id` | `{-term-id}` |
| `ReferenceError` | Unknown attribute: `msg.attr` | `{msg.attr}` |
| `ReferenceError` | No value on message (only attributes exist) | `{msg-id}` |
| `ReferenceError` | Unknown function: `FUNC()` | `{FUNC()}` |
| `TypeError` | Unsupported variable type passed as argument | `{$name}` |
| `TypeError` | Function is not callable | `{FUNC()}` |
| `RangeError` | Cyclic reference detected | `{???}` |
| `RangeError` | No default variant in select expression | `{???}` |
| `RangeError` | Excessive placeables (>100) — **fatal, terminates resolution** | Throws |

#### Two Error Handling Modes

**Mode 1 — Silent collection** (recommended for production):
```typescript
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
// errors array now contains any issues
// text contains best-effort output with {???} for failed parts
```

**Mode 2 — Throw on first error** (useful for development/testing):
```typescript
// No errors array → throws on first resolution error
const text = bundle.formatPattern(pattern, args);
// Throws ReferenceError, TypeError, or RangeError
```

#### Parse-Time vs. Resolution-Time Errors

- **Parse-time errors** (returned by `addResource()`): Syntax errors in FTL. These are per-message — one bad message does not break others.
- **Resolution-time errors** (from `formatPattern()`): Missing variables, broken references, type mismatches. These occur when formatting a specific pattern.

---

### §2.5 Custom Functions

Custom functions extend Fluent's formatting capabilities beyond the built-in `NUMBER()` and `DATETIME()`.

#### FluentFunction Type Signature

```typescript
type FluentFunction = (
  positional: Array<FluentValue>,
  named: Record<string, FluentValue>
) => FluentValue;
```

Where `FluentValue = FluentType<unknown> | string`.

Functions receive two arguments:
1. `positional` — positional arguments from the FTL call expression
2. `named` — named arguments as key-value pairs

#### Registering Custom Functions

```typescript
const bundle = new FluentBundle("en-US", {
  functions: {
    GREETING_TIME: (positional, named) => {
      const hour = new Date().getHours();
      if (hour < 12) return "morning";
      if (hour < 18) return "afternoon";
      return "evening";
    },
    SHOUT: (positional, named) => {
      const text = positional[0];
      return typeof text === "string" ? text.toUpperCase() : String(text).toUpperCase();
    },
  },
});
```

Corresponding FTL:
```ftl
greeting = Good {GREETING_TIME()}!
shout = {SHOUT("hello")}
```

#### Built-in Functions: NUMBER and DATETIME

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/builtins.ts

**NUMBER()** — formats numeric values using `Intl.NumberFormat`.

```typescript
function NUMBER(
  args: Array<FluentValue>,
  opts: Record<string, FluentValue>
): FluentValue
```

Allowed options (passed through to `Intl.NumberFormat`):
`unitDisplay`, `currencyDisplay`, `useGrouping`, `minimumIntegerDigits`, `minimumFractionDigits`, `maximumFractionDigits`, `minimumSignificantDigits`, `maximumSignificantDigits`

FTL usage:
```ftl
price = { NUMBER($amount, minimumFractionDigits: 2) }
```

**DATETIME()** — formats date/time values using `Intl.DateTimeFormat`.

```typescript
function DATETIME(
  args: Array<FluentValue>,
  opts: Record<string, FluentValue>
): FluentValue
```

Allowed options (passed through to `Intl.DateTimeFormat`):
`dateStyle`, `timeStyle`, `fractionalSecondDigits`, `dayPeriod`, `hour12`, `weekday`, `era`, `year`, `month`, `day`, `hour`, `minute`, `second`, `timeZoneName`

FTL usage:
```ftl
last-seen = Last seen: { DATETIME($date, dateStyle: "long") }
```

Both functions return `FluentNone` if the input argument is `FluentNone`, effectively propagating failures gracefully. They throw `TypeError` for completely invalid argument types.

---

### §2.6 @fluent/syntax Package (Parser/Serializer)

**Current version**: 0.19.0
**npm package**: `@fluent/syntax`
**Node.js requirement**: `^20.19 || ^22.12 || >=24`

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-syntax/README.md

The `@fluent/syntax` package provides a **full AST parser and serializer** for Fluent files. It is a separate package from `@fluent/bundle` and serves a different purpose.

#### When to Use Which

| Use Case | Package |
|----------|---------|
| Runtime message formatting in apps | `@fluent/bundle` |
| Linting/validating FTL files | `@fluent/syntax` |
| Programmatic FTL generation or modification | `@fluent/syntax` |
| Building editor tooling (syntax highlighting, etc.) | `@fluent/syntax` |
| Extracting message IDs from FTL files | `@fluent/syntax` |
| Converting between formats | `@fluent/syntax` |

#### FluentParser

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-syntax/src/parser.ts

```typescript
interface FluentParserOptions {
  withSpans?: boolean; // default: true
}

class FluentParser {
  constructor(options?: FluentParserOptions);
  parse(source: string): Resource;
  parseEntry(source: string): Entry;
}
```

- `parse(source)` — parses a complete FTL document into a `Resource` AST node
- `parseEntry(source)` — parses the first Message or Term from the input; returns `Junk` on failure
- `withSpans: true` — each AST node includes a `Span` with `start` and `end` byte offsets

**Example**:
```typescript
import { FluentParser } from "@fluent/syntax";

const parser = new FluentParser({ withSpans: true });
const resource = parser.parse(`
hello = Hello, world!
welcome = Welcome, {$name}!
`);

for (const entry of resource.body) {
  if (entry.type === "Message") {
    console.log(entry.id.name); // "hello", "welcome"
    console.log(entry.span?.start, entry.span?.end);
  }
}
```

#### FluentSerializer

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-syntax/src/serializer.ts

```typescript
interface FluentSerializerOptions {
  withJunk?: boolean; // default: false
}

class FluentSerializer {
  withJunk: boolean;
  constructor(options?: FluentSerializerOptions);
  serialize(resource: Resource): string;
  serializeEntry(entry: Entry, state?: number): string;
}
```

Additionally exported standalone functions:
```typescript
function serializeExpression(expr: Expression | Placeable): string;
function serializeVariantKey(key: Identifier | NumberLiteral): string;
```

**Example — round-trip parse and serialize**:
```typescript
import { FluentParser, FluentSerializer } from "@fluent/syntax";

const parser = new FluentParser();
const serializer = new FluentSerializer();

const resource = parser.parse(`hello = Hello, world!`);
const output = serializer.serialize(resource);
// → "hello = Hello, world!\n"
```

#### AST Types

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-syntax/src/ast.ts

The full AST hierarchy:

**Base**: `BaseNode` → `SyntaxNode` (adds optional `span: Span`)

**Top-level**:
- `Resource` — `body: Array<Entry>` where `Entry = Message | Term | Comment | GroupComment | ResourceComment | Junk`

**Messages and Terms**:
- `Message` — `id: Identifier`, `value: Pattern | null`, `attributes: Array<Attribute>`, `comment: Comment | null`
- `Term` — `id: Identifier`, `value: Pattern`, `attributes: Array<Attribute>`, `comment: Comment | null`
- `Attribute` — `id: Identifier`, `value: Pattern`

**Patterns**:
- `Pattern` — `elements: Array<PatternElement>` where `PatternElement = TextElement | Placeable`
- `TextElement` — `value: string`
- `Placeable` — `expression: Expression`

**Expressions**:
- `StringLiteral` — `value: string` (with `parse()` method for escape sequences)
- `NumberLiteral` — `value: string` (with `parse()` method)
- `MessageReference` — `id: Identifier`, `attribute: Identifier | null`
- `TermReference` — `id: Identifier`, `attribute: Identifier | null`, `arguments: CallArguments | null`
- `VariableReference` — `id: Identifier`
- `FunctionReference` — `id: Identifier`, `arguments: CallArguments`
- `SelectExpression` — `selector: InlineExpression`, `variants: Array<Variant>`

**Supporting**:
- `Variant` — `key: Identifier | NumberLiteral`, `value: Pattern`, `default: boolean`
- `CallArguments` — `positional: Array<InlineExpression>`, `named: Array<NamedArgument>`
- `NamedArgument` — `name: Identifier`, `value: Literal`
- `Identifier` — `name: string`

**Comments**:
- `Comment` — `content: string` (single `#`)
- `GroupComment` — `content: string` (double `##`)
- `ResourceComment` — `content: string` (triple `###`)

**Errors**:
- `Junk` — `content: string`, `annotations: Array<Annotation>`
- `Annotation` — `code: string`, `arguments: Array<unknown>`, `message: string`
- `Span` — `start: number`, `end: number`

---

### §2.7 Type Exports and TypeScript Usage

The `@fluent/bundle` package is written in TypeScript (migrated in v0.15.0, January 2020). Type declarations are published alongside the ES module build at `esm/index.d.ts`.

#### Key Types for Consumers

```typescript
import type {
  FluentValue,      // FluentType<unknown> | string
  FluentVariable,   // FluentValue | TemporalObject | string | number | Date
  FluentFunction,   // (positional: FluentValue[], named: Record<string, FluentValue>) => FluentValue
  TextTransform,    // (text: string) => string
  Message,          // { value: Pattern | null; attributes: Record<string, Pattern> }
} from "@fluent/bundle";
```

#### FluentType Hierarchy

```typescript
abstract class FluentType<T> {
  value: T;
  constructor(value: T);
  valueOf(): T;
  abstract toString(scope: Scope): string;
}

class FluentNone extends FluentType<string> {
  constructor(value?: string); // default: "???"
  toString(scope: Scope): string; // returns `{${this.value}}`
}

class FluentNumber extends FluentType<number> {
  opts: Intl.NumberFormatOptions;
  constructor(value: number, opts?: Intl.NumberFormatOptions);
  toString(scope?: Scope): string;
}

class FluentDateTime extends FluentType<number> {
  opts: Intl.DateTimeFormatOptions;
  constructor(
    value: number | Date | TemporalObject | FluentDateTime | FluentType<number>,
    opts?: Intl.DateTimeFormatOptions
  );
  static supportsValue(value: unknown): boolean;
  toNumber(): number;
  toString(scope?: Scope): string;
}
```

#### Temporal Support (v0.19.0+)

As of version 0.19.0, `FluentDateTime` supports TC39 Temporal objects:

```typescript
interface TemporalInstant {
  epochMilliseconds: number;
  toString(): string;
}

interface TemporalDateTypes {
  calendarId: string;
  toZonedDateTime?(timeZone: string): { epochMilliseconds: number };
  toString(): string;
}

interface TemporalPlainTime {
  hour: number;
  minute: number;
  second: number;
  toString(): string;
}

type TemporalObject = TemporalInstant | TemporalDateTypes | TemporalPlainTime;
```

`FluentVariable` accepts Temporal objects directly:
```typescript
type FluentVariable = FluentValue | TemporalObject | string | number | Date;
```

---

### §2.8 Version History and Breaking Changes

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/CHANGELOG.md

#### Major Version Milestones

| Version | Date | Key Change |
|---------|------|------------|
| 0.19.1 | 2025-04-02 | Fix FluentDateTime/FluentNumber primitive conversions |
| 0.19.0 | 2025-03-25 | Temporal support; Node.js >= 18 required |
| 0.18.0 | 2023-03-13 | Drop Node.js v12; documentation format update |
| 0.17.0 | 2021-09-13 | ESM-first: `"type": "module"` in esm/ directory; Node.js 12 minimum |
| 0.16.0 | 2020-07-01 | Rename `FluentArgument` → `FluentVariable`; remove compat builds; ES2018 target |
| 0.15.0 | 2020-01-23 | Codebase migrated to TypeScript; `FluentValue` type exported |
| 0.14.0 | 2019-07-30 | **Major API overhaul**: remove `addMessages`, `format`; add `formatPattern`, `FluentResource` constructor |
| 0.13.0 | 2019-07-25 | Package renamed from `fluent` to `@fluent/bundle` |
| 0.8.0 | 2018-08-20 | `MessageContext` → `FluentBundle`; `MessageArgument` → `FluentType` |

#### Critical Breaking Changes to Know

**v0.14.0 (the big API break)**:
- `FluentBundle.addMessages(source)` removed → use `addResource(new FluentResource(source))`
- `FluentBundle.format(msg, args, errors)` removed → use `formatPattern(getMessage(id).value, args, errors)`
- `FluentBundle.messages` property removed → use `getMessage()` and `hasMessage()`
- `FluentResource.fromString()` removed → use `new FluentResource(source)`
- `getMessage()` return shape changed to `{ value: Pattern | null, attributes: Record<string, Pattern> }`
- `formatPattern` **throws** if no errors array is provided (unlike old `format` which silently returned)

**v0.16.0**:
- `FluentArgument` type renamed to `FluentVariable`
- Compat builds removed — everything compiles to ES2018

**v0.15.1**:
- NUMBER and DATETIME builtins now accept ONLY specific formatting options (see Section §2.5 for allowed lists)

**v0.19.0**:
- Requires Node.js >= 18

---

### §2.9 Common Issues from GitHub

Source: https://github.com/projectfluent/fluent.js/issues

#### Open Issues (as of 2026-03-19)

| # | Title | Category | Impact |
|---|-------|----------|--------|
| 648 | Using Fluent with a lot of roots is slow | Performance | Scale issues when many FluentBundle instances are created |
| 646 | Usage example for SSR on Next.js | Documentation | No official SSR guidance |
| 628 | How can a variable be referenced in terms params? | API confusion | Terms parameterization not intuitive |
| 626 | Synchronising .ftl files and finding fluent IDs | Tooling | No built-in sync tooling |
| 625 | RFC: fluent babel plugin | Feature request | No build-time optimization |
| 624 | Support prefix identifier for message | Feature request | Namespace management |
| 615 | DOMLocalization setArgs method | Enhancement | DOM API limitations |
| 599 | Disabling unicode isolation per-placeable | Feature request | Fine-grained bidi control |
| 598 | Support ref forwarding in withLocalization | React | React pattern incompatibility |
| 597 | Support range versions of NUMBER/DATETIME | Feature request | Limited built-in formatting |
| 587 | Transform placeables (escape values) | Feature request | No placeable-level transform |
| 583 | Automatic Eastern Arabic numerals | Localization | Incomplete locale formatting |

#### Key Closed Issues

| # | Title | Impact |
|---|-------|--------|
| 376 | Migrate to TypeScript? | Resolved in v0.15.0 |
| 217 | Introduce FluentResource and addResource | Resolved in v0.14.0 |
| 208 | format() should accept a string identifier | Led to getMessage + formatPattern two-step API |
| 364 | Emit errors on FluentBundle instances | Led to current error array pattern |
| 222 | Rename MessageContext to FluentContext | Resolved as FluentBundle in v0.8.0 |

#### Patterns in Issues

1. **Performance at scale**: Multiple bundles (one per locale per component) creates overhead. No built-in bundle-sharing strategy.
2. **SSR gaps**: No official guidance for server-side rendering with Fluent.
3. **Terms confusion**: How terms interact with parameterization is a common source of questions.
4. **React integration friction**: `withLocalization` HOC does not support ref forwarding, pushing users toward hooks.
5. **Bidi control**: `useIsolating: true` is all-or-nothing; users want per-placeable control.

---

<!-- §3 START — @fluent/react and @fluent/langneg -->

## §3 @fluent/react and @fluent/langneg

> Research date: 2026-03-19
> Sources: GitHub wiki, source code, package.json, GitHub issues — all fetched via WebFetch

---

### §3.1 @fluent/react Overview and Version

**Current version:** 0.15.2 (from `fluent-react/package.json` on GitHub main branch)

**Description:** "Fluent bindings for React" — provides the React integration layer for Project Fluent, exposing translations via the React Context API and component patterns.

**License:** Apache-2.0

**Entry points:**
- CommonJS: `./index.js`
- ESM: `./esm/index.js`
- TypeScript definitions: `./esm/index.d.ts`

**Dependencies:**
- `@fluent/sequence`: ^0.8.0
- `cached-iterable`: ^0.3.0

**Peer dependencies:**
- `@fluent/bundle`: >=0.16.0
- `react`: >=16.8.0 (hooks support required)

**Node engine requirement:** ^20.19 || ^22.12 || >=24

**Complete public API exports** (from `src/index.ts`):

| Export | Type | Source module |
|--------|------|---------------|
| `ReactLocalization` | Class | `./localization.js` |
| `LocalizationProvider` | Component | `./provider.js` |
| `withLocalization` | HOC | `./with_localization.js` |
| `WithLocalizationProps` | TypeScript type | `./with_localization.js` |
| `Localized` | Component | `./localized.js` |
| `LocalizedProps` | TypeScript type | `./localized.js` |
| `MarkupParser` | Type | `./markup.js` |
| `useLocalization` | Hook | `./use_localization.js` |

**Installation (complete stack):**
```bash
npm install @fluent/bundle @fluent/langneg @fluent/react
```

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-react/package.json, https://github.com/projectfluent/fluent.js/blob/main/fluent-react/src/index.ts

---

### §3.2 LocalizationProvider

The `LocalizationProvider` component uses React's Context API to distribute translations throughout the component tree. Every `<Localized>` component and every `useLocalization()` call must be a descendant of a `<LocalizationProvider>`.

#### §3.2.1 Props and Configuration

The provider accepts an `l10n` prop (a `ReactLocalization` instance) in the wiki documentation pattern, but the React Tutorial shows a `bundles` prop pattern. Based on the source code and tutorial, the canonical usage is:

```tsx
import { LocalizationProvider } from "@fluent/react";

<LocalizationProvider l10n={l10n}>
  <App />
</LocalizationProvider>
```

Where `l10n` is a `ReactLocalization` instance. The provider makes this instance available to all descendant components via React Context.

**Key behavior:**
- Manages subscriptions for all nested `<Localized>` elements
- Establishes a fallback chain — if a translation is missing in one bundle, the system proceeds to the next bundle
- Context-based: eliminates manual prop-threading of translation objects

Source: https://github.com/projectfluent/fluent.js/wiki/LocalizationProvider

#### §3.2.2 Bundle Generation Pattern

Bundles can be generated synchronously (array) or lazily (generator). Both approaches yield `FluentBundle` instances:

**Array-based (eager, synchronous):**
```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

function generateBundles(currentLocales: string[]): FluentBundle[] {
  return currentLocales.map((locale) => {
    const bundle = new FluentBundle(locale);
    for (const file of FILES[locale]) {
      const resource = new FluentResource(file);
      bundle.addResource(resource);
    }
    return bundle;
  });
}
```

**Generator-based (lazy):**
```typescript
function* generateBundles(currentLocales: string[]): Generator<FluentBundle> {
  for (const locale of currentLocales) {
    const bundle = new FluentBundle(locale);
    for (const file of FILES[locale]) {
      const resource = new FluentResource(file);
      bundle.addResource(resource);
    }
    yield bundle;
  }
}
```

The generator pattern is preferable for large locale sets because fallback bundles are only created when needed (lazy evaluation). The `ReactLocalization` constructor wraps the iterable in `CachedSyncIterable.from()` so it is only iterated once and then cached.

Source: https://github.com/projectfluent/fluent.js/wiki/ReactLocalization, https://github.com/projectfluent/fluent.js/blob/main/fluent-react/src/localization.ts

#### §3.2.3 Async Bundle Loading

Translations are typically stored in `.ftl` files on the server and loaded asynchronously. The recommended pattern fetches all needed translations before passing them to the provider:

```typescript
async function getMessages(locale: string): Promise<string> {
  const url = `/static/locale/${locale}/content.ftl`;
  const response = await fetch(url);
  return await response.text();
}

async function createBundles(locales: string[]): Promise<FluentBundle[]> {
  return Promise.all(
    locales.map(async (locale) => {
      const messages = await getMessages(locale);
      const bundle = new FluentBundle(locale);
      bundle.addResource(new FluentResource(messages));
      return bundle;
    })
  );
}

// In your app initialization:
const bundles = await createBundles(negotiatedLocales);
const l10n = new ReactLocalization(bundles);
```

**Critical limitation:** As of v0.15.2, all translations (including fallback locales) must be fetched before rendering `<LocalizationProvider>`. There is no built-in support for async/lazy fallback locale loading during render. Future versions may add this capability.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Tutorial, https://github.com/projectfluent/fluent.js/wiki/ReactLocalization

---

### §3.3 Localized Component

The `<Localized>` component is the primary declarative API for displaying translated content. It wraps a React element and replaces its text content (and optionally attributes) with translations.

#### §3.3.1 Basic Usage

```tsx
import { Localized } from "@fluent/react";

function Greeting() {
  return (
    <Localized id="hello-world">
      <p>Hello, World!</p>
    </Localized>
  );
}
```

The child element (`<p>`) serves as the fallback — it is displayed if the translation for `"hello-world"` is not found in any bundle.

**Props (from `LocalizedProps` type):**

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `id` | `string` | Yes | Translation message identifier |
| `attrs` | `Record<string, boolean>` | No | Map of attribute names to translate |
| `vars` | `Record<string, FluentVariable>` | No | Variables passed to the translation |
| `elems` | `Record<string, ReactElement>` | No | React elements for DOM overlay markup |
| `children` | `ReactElement` | No | Fallback element |

**With variables:**
```tsx
<Localized id="welcome-user" vars={{ userName: currentUser.name }}>
  <p>Welcome, {currentUser.name}!</p>
</Localized>
```

Corresponding FTL:
```ftl
welcome-user = Welcome, { $userName }!
```

Source: https://github.com/projectfluent/fluent.js/wiki/Localized, https://github.com/projectfluent/fluent.js/wiki/React-Tutorial

#### §3.3.2 DOM Overlays

DOM overlays enable translations to contain markup that maps to real React components. This is the `elems` prop mechanism:

```tsx
<Localized
  id="create-account"
  elems={{
    confirm: <button onClick={createAccount}></button>,
    cancel: <Link to="/"></Link>,
  }}
>
  <p>{"<confirm>Create account</confirm> or <cancel>go back</cancel>."}</p>
</Localized>
```

Corresponding FTL:
```ftl
create-account = <confirm>Create account</confirm> or <cancel>go back</cancel>.
```

**How it works internally:**
1. `@fluent/react` parses the translation using a `<template>` element, creating an inert `DocumentFragment`
2. Unknown elements (like `<confirm>`, `<cancel>`) become `HTMLUnknownElement` instances
3. The library matches these by name against the `elems` prop keys
4. Provided React elements are cloned with the translated text content as children

**Result:**
```html
<p>
  <button onClick={createAccount}>Create account</button> or
  <Link to="/">go back</Link>.
</p>
```

**Custom markup parser (for SSR / React Native):**
When `<template>` is unavailable (server-side rendering, React Native), pass a custom `parseMarkup` function to `ReactLocalization`:

```typescript
type MarkupParser = (str: string) => Array<Node>;
// Each Node must have: nodeName, textContent

const l10n = new ReactLocalization(bundles, customParseMarkup);
```

For SSR, `jsdom` or `cheerio` can be used as the parser backend.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Overlays

#### §3.3.3 Attribute Mapping

The `attrs` prop controls which FTL message attributes are applied to the wrapped element:

```tsx
<Localized id="my-link" attrs={{ title: true }}>
  <a href="/something" title="Link to something">
    Something
  </a>
</Localized>
```

Corresponding FTL:
```ftl
my-link = Something
    .title = Link to something
```

**Key rule:** Translated attributes are ONLY applied when explicitly declared as `true` in the `attrs` object. This is a security measure — translations cannot inject arbitrary attributes onto DOM elements.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Tutorial

---

### §3.4 useLocalization Hook

The `useLocalization` hook provides imperative access to translations within functional components. It returns the current `ReactLocalization` instance from context.

```typescript
import { useLocalization } from "@fluent/react";

function AlertButton() {
  const { l10n } = useLocalization();

  const handleClick = () => {
    alert(l10n.getString("confirm-delete"));
  };

  return <button onClick={handleClick}>Delete</button>;
}
```

**Return value:** `{ l10n: ReactLocalization }`

**Primary use case:** Imperative translation formatting — `window.alert()`, `window.confirm()`, logging, validation messages, or any scenario where JSX rendering via `<Localized>` is not possible.

**`getString` method signature:**
```typescript
getString(
  id: string,
  vars?: Record<string, FluentVariable> | null,
  fallback?: string
): string
```

If the translation is not found, `getString` returns the `fallback` parameter, or the `id` itself if no fallback is provided.

Source: https://github.com/projectfluent/fluent.js/wiki/useLocalization

---

### §3.5 ReactLocalization Class

The `ReactLocalization` class is the core engine of `@fluent/react`. It stores, caches, and queries a sequence of `FluentBundle` instances.

**Constructor:**
```typescript
constructor(
  bundles: Iterable<FluentBundle>,
  parseMarkup?: MarkupParser | null,   // defaults to createParseMarkup()
  reportError?: (error: Error) => void  // defaults to console.warn
)
```

**Properties:**
| Property | Type | Description |
|----------|------|-------------|
| `bundles` | `Iterable<FluentBundle>` | Bundle sequence (wrapped in `CachedSyncIterable`) |
| `parseMarkup` | `MarkupParser \| null` | Parser for DOM overlay markup |
| `reportError` | `(error: Error) => void` | Error reporting callback |

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `getBundle` | `(id: string) => FluentBundle \| null` | Find first bundle containing message `id` |
| `getString` | `(id: string, vars?: Record<string, FluentVariable> \| null, fallback?: string) => string` | Format message to string; returns fallback or id if not found |
| `getElement` | `(sourceElement: ReactElement, id: string, args?: { vars?, elems?, attrs? }) => ReactElement` | Format message into a React element with overlays and attributes |
| `areBundlesEmpty` | `() => boolean` | Check if bundle iterable is exhausted/empty |

**Bundle resolution:** The class iterates through bundles in order (representing the user's locale preference chain). The first bundle that contains the requested message `id` wins. This enables graceful degradation — missing translations fall through to the next locale.

**Caching:** The constructor wraps the bundle iterable with `CachedSyncIterable.from()`, ensuring the potentially expensive iteration (especially from generators) only happens once.

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-react/src/localization.ts

---

### §3.6 Locale Switching Patterns

To switch locales at runtime, create a new `ReactLocalization` instance with re-ordered bundles and update the provider. The typical React pattern uses state:

```tsx
import React, { useState, useCallback } from "react";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization, LocalizationProvider } from "@fluent/react";

const AVAILABLE_LOCALES = ["en-US", "fr", "de"];

function generateBundles(locales: string[]): FluentBundle[] {
  return locales.map((locale) => {
    const bundle = new FluentBundle(locale);
    bundle.addResource(new FluentResource(MESSAGES[locale]));
    return bundle;
  });
}

function App() {
  const [currentLocales, setCurrentLocales] = useState(() =>
    negotiateLanguages(navigator.languages, AVAILABLE_LOCALES, {
      defaultLocale: "en-US",
    })
  );

  const l10n = new ReactLocalization(generateBundles(currentLocales));

  const switchLocale = useCallback((locale: string) => {
    const negotiated = negotiateLanguages(
      [locale],
      AVAILABLE_LOCALES,
      { defaultLocale: "en-US" }
    );
    setCurrentLocales(negotiated);
  }, []);

  return (
    <LocalizationProvider l10n={l10n}>
      <LocaleSwitcher onSwitch={switchLocale} />
      <MainContent />
    </LocalizationProvider>
  );
}
```

When `currentLocales` state changes, React re-renders the provider with a new `ReactLocalization` instance, and all `<Localized>` components automatically re-translate.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Bindings, https://github.com/projectfluent/fluent.js/wiki/React-Tutorial

---

### §3.7 @fluent/langneg Overview

**Current version:** 0.7.0

**Description:** "Language Negotiation API for Fluent"

**License:** Apache-2.0

**Dependencies:** None (zero runtime dependencies)

**Node engine requirement:** ^20.19 || ^22.12 || >=24

**Exports (from `src/index.ts`):**

| Export | Type | Description |
|--------|------|-------------|
| `negotiateLanguages` | Function | Main negotiation algorithm |
| `acceptedLanguages` | Function | Parse HTTP Accept-Language header |
| `filterMatches` | Function | Low-level matching engine |

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/package.json, https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/index.ts

#### §3.7.1 negotiateLanguages()

The primary function that matches user-requested locales against application-available locales.

**Full TypeScript signature:**
```typescript
function negotiateLanguages(
  requestedLocales: Readonly<Array<string>>,
  availableLocales: Readonly<Array<string>>,
  options?: {
    strategy?: "filtering" | "matching" | "lookup";
    defaultLocale?: string;
  }
): Array<string>
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `requestedLocales` | `Readonly<Array<string>>` | User's preferred locales (e.g., `navigator.languages`) |
| `availableLocales` | `Readonly<Array<string>>` | Locales the app supports |
| `options.strategy` | `"filtering" \| "matching" \| "lookup"` | Negotiation algorithm (default: `"filtering"`) |
| `options.defaultLocale` | `string` | Fallback locale; appended if not already in results |

**Return value:** `Array<string>` — locale identifiers ordered by preference.

**Implementation logic:**
1. Calls `filterMatches()` with the chosen strategy
2. For `"lookup"` strategy: throws `Error` if `defaultLocale` is undefined; pushes `defaultLocale` if no matches found
3. For other strategies: appends `defaultLocale` to results if provided and not already present

**Usage example:**
```typescript
import { negotiateLanguages } from "@fluent/langneg";

const supported = negotiateLanguages(
  navigator.languages,        // e.g., ["de-DE", "fr-FR"]
  ["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"],
  { defaultLocale: "en-US" }
);
```

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/negotiate_languages.ts, https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/README.md

#### §3.7.2 Strategies

Given requested `["de-DE", "fr-FR"]` and available `["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"]`:

**Filtering (default):** Returns ALL available locales that match ANY requested locale. Most permissive.
- Result: `["de-DE", "de", "fr", "fr-CA"]`
- Use when: you want maximum coverage with multiple fallbacks per language

**Matching:** Returns the BEST single match for EACH requested locale. One result per request.
- Result: `["de-DE", "fr"]`
- Use when: you want one best-fit per user preference

**Lookup:** Returns the SINGLE best match across ALL requested locales. Most restrictive.
- Result: `["de-DE"]`
- Use when: you need exactly one locale (e.g., selecting a date format library)
- **Requirement:** `defaultLocale` MUST be provided or the function throws an `Error`

**Likely subtags:** The module includes minimal likely-subtags data to resolve generic locales. Requesting `"en"` can match both `"en-GB"` and `"en-US"`.

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/README.md

#### §3.7.3 BCP 47 Compliance

All locale identifiers in `@fluent/langneg` follow BCP 47 (IETF language tag standard):
- Language: `en`, `fr`, `de`
- Language + region: `en-US`, `fr-CA`, `de-DE`
- Language + script: `sr-Latn`, `zh-Hans`

The negotiation algorithms perform subtag matching — `"de-DE"` will match `"de"` as a fallback. Input locales are coerced to strings via `Array.from(...).map(String)`.

**acceptedLanguages() helper:**
For server-side usage, parse the HTTP `Accept-Language` header:

```typescript
import { acceptedLanguages } from "@fluent/langneg";

// Parse: "fr-CH, fr;q=0.9, en;q=0.8, de;q=0.7, *;q=0.5"
const locales = acceptedLanguages(req.headers["accept-language"]);
// Result: ["fr-CH", "fr", "en", "de", "*"] (sorted by q-value)
```

**Implementation:** Parses quality values (`q=`) from the header, sorts by descending quality while preserving order for equal weights. Throws `TypeError` if argument is not a string.

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/accepted_languages.ts

---

### §3.8 Complete Integration Pattern (Full Example)

A production-ready setup combining all packages:

```typescript
// l10n.ts — Localization setup module
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";

const AVAILABLE_LOCALES = ["en-US", "fr", "it"];
const DEFAULT_LOCALE = "en-US";

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  return response.text();
}

function createBundle(locale: string, messages: string): FluentBundle {
  const bundle = new FluentBundle(locale);
  const resource = new FluentResource(messages);
  const errors = bundle.addResource(resource);
  if (errors.length) {
    // Log but do not throw — partial translations are acceptable
    errors.forEach((e) => console.warn(`FTL parse error in ${locale}:`, e));
  }
  return bundle;
}

export async function initLocalization(): Promise<ReactLocalization> {
  const negotiated = negotiateLanguages(
    navigator.languages,
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const bundles = await Promise.all(
    negotiated.map(async (locale) => {
      const messages = await fetchMessages(locale);
      return createBundle(locale, messages);
    })
  );

  return new ReactLocalization(bundles);
}
```

```tsx
// App.tsx — Root component
import React, { useState, useEffect } from "react";
import { LocalizationProvider, Localized } from "@fluent/react";
import { ReactLocalization } from "@fluent/react";
import { initLocalization } from "./l10n";

function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    initLocalization().then(setL10n);
  }, []);

  if (!l10n) {
    return <div>Loading...</div>;
  }

  return (
    <LocalizationProvider l10n={l10n}>
      <Localized id="app-title">
        <h1>My Application</h1>
      </Localized>
    </LocalizationProvider>
  );
}
```

```ftl
# locales/en-US/messages.ftl
app-title = My Application
welcome-user = Welcome, { $userName }!
confirm-delete = Are you sure you want to delete { $itemName }?
create-account = <confirm>Create account</confirm> or <cancel>go back</cancel>.

# locales/fr/messages.ftl
app-title = Mon Application
welcome-user = Bienvenue, { $userName } !
confirm-delete = Voulez-vous vraiment supprimer { $itemName } ?
create-account = <confirm>Créer un compte</confirm> ou <cancel>revenir</cancel>.
```

Source: Composite example based on patterns from https://github.com/projectfluent/fluent.js/wiki/React-Tutorial, https://github.com/projectfluent/fluent.js/wiki/ReactLocalization

---

### §3.9 SSR Considerations

Server-side rendering with `@fluent/react` has known challenges:

1. **No `<template>` element on server:** The DOM overlay system relies on `<template>` to parse HTML in translations. On the server (Node.js), this element does not exist. You MUST provide a custom `parseMarkup` function using `jsdom` or `cheerio`:

```typescript
import { JSDOM } from "jsdom";

function parseMarkup(str: string): Node[] {
  const dom = new JSDOM(`<body>${str}</body>`);
  return Array.from(dom.window.document.body.childNodes);
}

const l10n = new ReactLocalization(bundles, parseMarkup);
```

2. **Locale detection on server:** Use `acceptedLanguages()` to parse the HTTP `Accept-Language` header instead of `navigator.languages`:

```typescript
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";

const requested = acceptedLanguages(req.headers["accept-language"] || "");
const supported = negotiateLanguages(requested, AVAILABLE_LOCALES, {
  defaultLocale: "en-US",
});
```

3. **No async fallback:** All bundles must be ready before rendering. This means server-side code must pre-load all translations synchronously or await them before calling `renderToString()`.

4. **Open issue #646:** There is an open issue requesting documented SSR examples for Next.js and other React-based frameworks, indicating this is an area where the community needs more guidance.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Overlays, https://github.com/projectfluent/fluent.js/issues (issue #646)

---

### §3.10 React-specific Issues

Based on GitHub issues analysis (https://github.com/projectfluent/fluent.js/issues?q=is%3Aissue+react):

#### Open Issues (React-specific)

| # | Issue | Impact |
|---|-------|--------|
| #646 | No SSR documentation for Next.js | Developers struggle with server-side setup |
| #598 | `withLocalization` does not support ref forwarding | Cannot use refs on wrapped components |
| #571 | Insufficient `<Localized>` documentation | Developers confused about prop behavior |
| #550 | Cannot use `elems` without `children` | Limits flexibility of overlay API |
| #538 | React Overlays do not work as documented | Documentation-implementation mismatch |
| #537 | Tutorial has compile-time errors | Onboarding friction |
| #522 | No RTL/LTR direction support | Missing bidirectional text handling |
| #500 | No `setL10n` in `useLocalization` | Cannot trigger re-localization imperatively |
| #498 | Unclear behavior with text node children | API ambiguity |

#### Resolved Issues Worth Noting

| # | Issue | Resolution |
|---|-------|------------|
| #546 | React Native compatibility | Works but needs custom `parseMarkup` (no DOM) |
| #519 | No error reporting for missing IDs | Added `reportError` callback to `ReactLocalization` constructor |
| #473 | Rules of Hooks violations | Fixed in library code |
| #476 | Loading `.ftl` files in Create React App | Requires custom webpack config or fetch-based loading |

Source: https://github.com/projectfluent/fluent.js/issues?q=is%3Aissue+react

---

### §3.11 File Organization Best Practices

#### Recommended directory structure

```
src/
├── l10n/
│   ├── index.ts              # initLocalization(), locale switching
│   └── bundles.ts            # Bundle generation logic
├── components/
│   └── LocalizedApp.tsx      # LocalizationProvider wrapper
public/
└── locales/
    ├── en-US/
    │   ├── main.ftl          # Core UI messages
    │   ├── errors.ftl        # Error messages
    │   └── settings.ftl      # Settings page messages
    ├── fr/
    │   ├── main.ftl
    │   ├── errors.ftl
    │   └── settings.ftl
    └── de/
        ├── main.ftl
        ├── errors.ftl
        └── settings.ftl
```

#### Key principles

1. **One FTL file per feature/page per locale** — keeps translations manageable and enables code-splitting
2. **Flat structure for small apps** — `locales/en-US.ftl`, `locales/fr.ftl` is acceptable for fewer than 50 messages
3. **Multiple resources per bundle** — a single `FluentBundle` can hold multiple `FluentResource` instances via repeated `addResource()` calls
4. **Semantic message IDs** — use `component-element-action` pattern: `login-button-submit`, `header-nav-home`
5. **Co-locate fallback strings** — the English text inside `<Localized>` children serves as both fallback and source for translators grepping the codebase

Source: https://github.com/projectfluent/fluent.js/wiki/React-Tutorial

---

<!-- CONSOLIDATED ANTI-PATTERNS -->

## Consolidated Anti-Patterns

All anti-patterns collected from the three research areas, numbered AP-01 through AP-16.

---

### AP-01: Passing FTL strings directly to addResource

**Source:** §2 (Bundle API)

```typescript
// WRONG — addResource expects a FluentResource, not a string
bundle.addResource(`hello = Hello`); // TypeError

// CORRECT
bundle.addResource(new FluentResource(`hello = Hello`));
```

This mistake is common among developers migrating from the pre-0.14.0 API where `addMessages()` accepted raw strings.

---

### AP-02: Ignoring the two-step getMessage + formatPattern flow

**Source:** §2 (Bundle API)

```typescript
// WRONG — there is no bundle.format() method (removed in v0.14.0)
bundle.format("welcome", { name: "Anna" }); // TypeError

// CORRECT — two-step: get message, then format its pattern
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  bundle.formatPattern(msg.value, { name: "Anna" });
}
```

---

### AP-03: Not checking for null value

**Source:** §2 (Bundle API)

```typescript
// DANGEROUS — messages can have null values (attributes-only messages)
const msg = bundle.getMessage("login-input")!;
bundle.formatPattern(msg.value!, args); // Runtime error if value is null

// CORRECT — always check
const msg = bundle.getMessage("login-input");
if (msg?.value) {
  bundle.formatPattern(msg.value, args);
}
```

---

### AP-04: Omitting the errors array in production

**Source:** §2 (Bundle API)

```typescript
// DANGEROUS — throws on first resolution error, crashing the app
const text = bundle.formatPattern(pattern, args);

// CORRECT for production — collect errors gracefully
const errors: Error[] = [];
const text = bundle.formatPattern(pattern, args, errors);
```

---

### AP-05: Trying to access terms via getMessage

**Source:** §2 (Bundle API)

```typescript
// WRONG — terms (prefixed with -) are not accessible via getMessage
const term = bundle.getMessage("-brand-name"); // returns undefined

// Terms are resolved automatically when referenced in messages:
// welcome = Welcome to {-brand-name}!
// There is NO public API to directly format a term.
```

---

### AP-06: Creating FluentResource per message

**Source:** §2 (Bundle API)

```typescript
// WRONG — inefficient, creates parser overhead per message
for (const [id, text] of Object.entries(messages)) {
  bundle.addResource(new FluentResource(`${id} = ${text}`));
}

// CORRECT — batch all messages into one resource
const ftl = Object.entries(messages)
  .map(([id, text]) => `${id} = ${text}`)
  .join("\n");
bundle.addResource(new FluentResource(ftl));
```

---

### AP-07: Using @fluent/syntax for runtime formatting

**Source:** §2 (Bundle API)

```typescript
// WRONG — @fluent/syntax is a tooling library, too heavy for runtime
import { FluentParser } from "@fluent/syntax";
const ast = new FluentParser().parse(ftlSource);
// ...manually walking the AST to format messages

// CORRECT — use @fluent/bundle for runtime
import { FluentBundle, FluentResource } from "@fluent/bundle";
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftlSource));
```

`@fluent/bundle` includes its own optimized runtime parser. `@fluent/syntax` is for tooling: linting, programmatic FTL generation, editor integrations.

---

### AP-08: Passing unsupported types as variables

**Source:** §2 (Bundle API)

```typescript
// WRONG — objects, arrays, booleans are not valid FluentVariable types
bundle.formatPattern(pattern, { user: { name: "Anna" } }); // TypeError

// CORRECT — only string, number, Date, FluentType, or Temporal objects
bundle.formatPattern(pattern, { name: "Anna", count: 5, date: new Date() });
```

---

### AP-09: Not handling addResource errors

**Source:** §2 (Bundle API)

```typescript
// WRONG — silently ignoring parse errors
bundle.addResource(new FluentResource(userProvidedFtl));

// CORRECT — check and log errors
const errors = bundle.addResource(new FluentResource(userProvidedFtl));
if (errors.length > 0) {
  errors.forEach(e => console.error("FTL parse error:", e.message));
}
```

---

### AP-10: Disabling useIsolating without understanding bidi

**Source:** §2 (Bundle API)

```typescript
// RISKY — disabling isolation marks can break RTL/LTR mixed text
const bundle = new FluentBundle("ar", { useIsolating: false });

// ONLY disable when you are certain your content is unidirectional
// or when testing and the Unicode marks interfere with assertions
const bundle = new FluentBundle("en-US", { useIsolating: false });
```

---

### AP-11: Post-translation string manipulation

**Source:** §3 (React)

NEVER concatenate or manipulate translated strings. Treat translation output as opaque. Let FTL handle all string composition through placeables and selectors.

---

### AP-12: Changing IDs without semantic change

**Source:** §3 (React)

Translation IDs are contracts with translators. ALWAYS change the ID when the meaning of the message changes — this signals translators that re-translation is needed.

---

### AP-13: Missing `attrs` declaration

**Source:** §3 (React)

Translated attributes are ONLY applied when explicitly whitelisted in `attrs={{ attributeName: true }}`. Forgetting this causes attributes to silently not translate.

---

### AP-14: Creating ReactLocalization in render

**Source:** §3 (React)

Avoid creating a new `ReactLocalization` instance on every render. Use `useState` or `useMemo` to prevent unnecessary re-initialization and re-rendering of all localized components.

---

### AP-15: Not handling addResource errors (React context)

**Source:** §3 (React)

`bundle.addResource()` returns an array of errors for malformed FTL. Ignoring these leads to silent translation failures. (Same as AP-09 but called out separately in React integration context.)

---

### AP-16: Using `withLocalization` instead of `useLocalization`

**Source:** §3 (React)

The HOC does not support ref forwarding (issue #598). Prefer the `useLocalization` hook in functional components.

---

*End of vooronderzoek. This document is the single reference for all skill creation agents.*
