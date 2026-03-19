# Part 1: FTL Specification and Syntax

> **Source version**: Fluent Syntax 1.0 (released April 2019)
> **Sources fetched**: 2026-03-19
> **All claims sourced from**: projectfluent.org (syntax guide), github.com/projectfluent/fluent (EBNF spec)

---

## 1.1 Design Philosophy

Project Fluent is a localization system created by Mozilla, designed for "natural-sounding translations." Its core mission statement: **"Translators should be able to use the entire expressive power of their language without asking developers for permission."**

Source: https://projectfluent.org

### Asymmetric Localization

The defining principle of Fluent is **asymmetric localization**. Traditional i18n systems (gettext, Java properties, iOS .strings) enforce a one-to-one mapping between source strings and translations. Fluent breaks this constraint:

- A simple string in English can map to a complex multi-variant translation in another language.
- Natural-sounding translations with genders and grammatical cases are used only when necessary.
- Expressiveness is not limited by the grammar of the source language.
- Locale-specific logic is isolated so it does not leak to other locales.

As stated on projectfluent.org: "Software localization has been dominated by an outdated paradigm: translations that map one-to-one to the English copy. The grammar of the source language imposes limits on the expressiveness of the translation." Fluent eliminates this limitation.

### Key Design Principles

1. **Progressive Enhancement**: Each language develops independently without impacting others. English may use a simple string while Polish uses a complex selector with grammatical cases.
2. **Simplicity for simple cases**: The basic `key = value` syntax is as simple as any properties file. Complexity is added only when needed.
3. **Fully-featured**: Date, time, and number formatting. Plural categories. Bidirectionality support. Custom formatters. Robust error handling.
4. **Terms system**: Reusable message fragments that enforce consistency across translations (brand names, product terms).

Source: https://projectfluent.org

---

## 1.2 Message Syntax

The basic unit of translation in Fluent is called a **message**. Messages are containers for information, consisting of an identifier, an equals sign, and a value (called a **pattern**).

Source: https://projectfluent.org/fluent/guide/hello.html

### Simple Messages

```ftl
hello = Hello, world!
```

The text value begins at the first non-blank character after the `=` sign. The identifier is what developers use to reference the message in code.

### Multiline Messages

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

### Whitespace and Dedentation in Multiline

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

### Message References

One message can be embedded inside another using placeable syntax:

```ftl
menu-save = Save
help-menu-save = Click { menu-save } to save the file.
```

This keeps translations consistent and makes maintenance easier.

Source: https://projectfluent.org/fluent/guide/references.html

### Formal Grammar (Messages)

From the EBNF specification:

```ebnf
Message ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*
```

A message MUST start with a letter. It may contain letters, digits, underscores, and hyphens. A message MUST have either a value (Pattern), attributes, or both.

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

## 1.3 Placeables and Variables

Placeables are syntax elements denoted by curly braces `{` and `}` that incorporate dynamic content into text. They enable variable interpolation, message/term references, and function calls.

Source: https://projectfluent.org/fluent/guide/placeables.html

### Variable References

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

### Automatic Number and Date Formatting

Numbers and dates are automatically formatted according to the user's locale:

```ftl
time-elapsed = Time elapsed: { $duration }s.
```

With `$duration = 12345.678`:
- American English: `Time elapsed: 12,345.678s.`
- South African English: `Time elapsed: 12 345,678s.`

Source: https://projectfluent.org/fluent/guide/variables.html

### Explicit Formatting Control

Built-in functions allow overriding automatic formatting:

```ftl
time-elapsed = Time elapsed: { NUMBER($duration, maximumFractionDigits: 0) }s.
```

This produces `Time elapsed: 12,345s.` regardless of the default fraction digit count.

### Formal Grammar (Placeables and Variables)

```ebnf
inline_placeable  ::= "{" blank? (SelectExpression | InlineExpression) blank? "}"
VariableReference ::= "$" Identifier
InlineExpression  ::= StringLiteral | NumberLiteral | FunctionReference
                    | MessageReference | TermReference | VariableReference
                    | inline_placeable
```

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

## 1.4 Select Expressions and Plurals

Selectors allow defining multiple translation variants based on a runtime value. One variant MUST be marked as the default with the `*` prefix.

Source: https://projectfluent.org/fluent/guide/selectors.html

### Basic Select Expression

```ftl
emails =
    { $unreadEmails ->
        [one] You have one unread email.
       *[other] You have { $unreadEmails } unread emails.
    }
```

The selector expression (`$unreadEmails`) is evaluated, and the matching variant key is selected. If no key matches, the default variant (marked with `*`) is used.

### Selector Types

1. **String selectors**: Match directly against variant keys as string values.
2. **Numeric selectors**: Match either exact number values OR CLDR plural categories.

The CLDR plural categories are: `zero`, `one`, `two`, `few`, `many`, `other`. Which categories a language uses depends on its plural rules. English uses only `one` and `other`.

### Formatted Number Selectors

When numbers need specific formatting in the selector, apply formatting options directly:

```ftl
your-score =
    { NUMBER($score, minimumFractionDigits: 1) ->
        [0.0]   You scored zero points. What happened?
       *[other] You scored { NUMBER($score, minimumFractionDigits: 1) } points.
    }
```

The formatted number determines which CLDR category applies. This may differ from the unformatted value's category.

### Ordinal Plurals

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

### Formal Grammar (Selectors)

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

## 1.5 Terms and References

Terms are specialized constructs that begin with a hyphen (`-`) and can ONLY be used as references in other messages. They cannot be retrieved directly by the localization runtime.

Source: https://projectfluent.org/fluent/guide/terms.html

### Basic Term Syntax

```ftl
-brand-name = Firefox

about = About { -brand-name }.
update-successful = { -brand-name } has been updated.
```

### Key Differences from Messages

| Aspect | Messages | Terms |
|--------|----------|-------|
| Prefix | None (starts with letter) | `-` (hyphen) |
| Runtime access | Can be retrieved directly | Cannot be retrieved directly |
| Purpose | User-facing text | Shared vocabulary / brand consistency |
| Variable data | Received from application | Received from referencing messages |

### Parameterized Terms

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

### Term Attributes for Grammatical Metadata

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

### Formal Grammar (Terms)

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

## 1.6 Attributes

Attributes allow multiple translatable strings to be associated with a single message or term, using the `.attribute` syntax on indented continuation lines.

Source: https://projectfluent.org/fluent/guide/attributes.html

### Attribute Syntax

```ftl
login-input = Predefined value
    .placeholder = email@example.com
    .aria-label = Login input value
    .title = Type your login email
```

This groups four related strings under one message identifier. Localization tools can display and manage them together.

### Use Cases

UI elements often contain multiple translatable strings per widget. An HTML form input may have a value, a `placeholder` attribute, an `aria-label` attribute, and a `title` attribute. Grouping them under one message improves maintainability.

### Attributes on Terms

Terms can also have attributes. Term attributes are **private** — they cannot be retrieved by the localization runtime and can ONLY be used as selectors (see section 1.5 for the gender example).

### Value-less Messages with Attributes

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

### Formal Grammar (Attributes)

```ebnf
Attribute         ::= line_end blank? "." Identifier blank_inline? "=" blank_inline? Pattern
AttributeAccessor ::= "." Identifier
```

An attribute starts on a new line, is indented, and begins with a dot (`.`) followed by an identifier.

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

## 1.7 Comments

Comments in Fluent use the `#` character and come in three levels, each serving a different organizational purpose.

Source: https://projectfluent.org/fluent/guide/comments.html

### Comment Levels

| Syntax | Name | Scope | Binding |
|--------|------|-------|---------|
| `#` | Comment | Message-level | Bound to the next message if directly above it |
| `##` | Group Comment | Section-level | Always standalone; divides file into groups |
| `###` | Resource Comment | File-level | Always standalone; describes entire file |

### Complete Example

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

### Comment Binding Rules

- A single-hash comment (`#`) placed **directly above** a message (no blank line between them) is considered part of that message. Localization tools present the comment together with the message.
- If there IS a blank line between a `#` comment and a message, the comment is standalone.
- `##` and `###` comments are ALWAYS standalone and never bind to messages.

### Formal Grammar (Comments)

```ebnf
CommentLine ::= ("###" | "##" | "#") ("\u0020" comment_char*)? line_end
comment_char ::= any_char - line_end
```

Note: The space after `#` / `##` / `###` is significant. A comment line is either the hash characters alone or the hash characters followed by a single space and then comment text. This means `#comment` (no space) is NOT valid — it must be `# comment`.

Source: https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf

---

## 1.8 Built-in Functions (NUMBER, DATETIME)

Fluent provides two built-in functions for explicit formatting of numeric and date/time values. In most cases, Fluent automatically selects the right formatter for the argument and formats it appropriately for the user's locale. The built-in functions give translators control when automatic formatting is insufficient.

Source: https://projectfluent.org/fluent/guide/builtins.html

### NUMBER()

Formats a number to a string in a given locale. Accepts the following named arguments (based on Intl.NumberFormat options):

```ftl
emails = You have { NUMBER($unreadEmails) } unread emails.

score = Your score: { NUMBER($score, minimumFractionDigits: 1, maximumFractionDigits: 1) }.
```

Known arguments:
- `minimumFractionDigits` — Minimum number of fraction digits
- `maximumFractionDigits` — Maximum number of fraction digits
- `type` — Can be set to `"ordinal"` for ordinal plural matching (see section 1.4)
- `minimumIntegerDigits` — Minimum number of integer digits
- `useGrouping` — Whether to use grouping separators (thousands)

The NUMBER function can be used in two contexts:
1. **In placeables** — for display formatting: `{ NUMBER($count, minimumFractionDigits: 2) }`
2. **In selectors** — for plural category determination: `{ NUMBER($count, type: "ordinal") -> ... }`

### DATETIME()

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

### Formal Grammar (Functions)

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

## 1.9 Custom Functions

The Fluent specification explicitly supports extensibility. From the documentation: "The list of available functions is extensible and environments may want to introduce additional functions, designed to aid localizers writing translations targeted for such environments."

Source: https://projectfluent.org/fluent/guide/functions.html

### What the Specification Defines

The FTL syntax itself does NOT define how custom functions are registered — that is implementation-specific. What FTL defines is:

1. Functions are called with `FUNCTION_NAME(arguments)` syntax inside placeables.
2. Arguments can be positional (`InlineExpression`) or named (`NamedArgument`).
3. Named argument values MUST be string literals or number literals (not variable references).

### Registration in @fluent/bundle (JavaScript)

Custom functions are registered when creating a `FluentBundle` instance. The function receives positional arguments and a named arguments object. This is covered in detail in the @fluent/bundle API documentation, not in the FTL syntax guide itself.

**Documentation gap**: The official syntax guide at projectfluent.org does NOT provide detailed guidance on implementing custom functions. It only states the extensibility principle. Implementation details must be sourced from the individual runtime libraries (@fluent/bundle for JS, fluent-rs for Rust, etc.).

---

## 1.10 Whitespace and Indentation Rules

Whitespace handling in Fluent is precise and has several rules that are critical to understand.

Source: https://projectfluent.org/fluent/guide/multiline.html, EBNF spec

### Core Rules

1. **Only spaces (U+0020) count as indentation.** Tabs are NOT indentation — they are treated as regular text characters.
2. **Continuation lines MUST be indented by at least one space** to be part of the preceding pattern.
3. **Common indent is stripped (dedented)** from all indented lines in a multiline pattern.
4. **The first line** (on the same line as `=`) is NOT considered indented and is excluded from dedentation calculations.
5. **Leading blank lines** (between `=` and the first content line) are ignored.
6. **Trailing blank lines** after the last content line are ignored.
7. **Interior blank lines** (with content both before and after) are preserved.

### Formal Whitespace Productions

```ebnf
blank_inline ::= "\u0020"+
line_end     ::= "\u000D\u000A" | "\u000A" | EOF
blank_block  ::= (blank_inline? line_end)+
blank        ::= (blank_inline | line_end)+
```

### Preserving Leading Whitespace

To force leading spaces in output, use quoted text:

```ftl
blank-is-preserved = {"    "}This message starts with 4 spaces.
```

Without the quoted text placeable, the leading spaces would be stripped by the dedentation algorithm.

### Block Text vs. Inline Text

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

## 1.11 Key Differences from ICU MessageFormat

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

## 1.12 Edge Cases and Gotchas

### 1. Special Characters at Line Start

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

### 2. Curly Braces in Text

Literal `{` and `}` characters MUST be escaped using quoted text placeables:

```ftl
opening-brace = This message features an opening curly brace: {"{"}.
closing-brace = This message features a closing curly brace: {"}"}.
```

Source: https://projectfluent.org/fluent/guide/special.html, EBNF `special_text_char ::= "{" | "}"`

### 3. Escape Sequences (Only in Quoted Text)

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

### 4. Junk Handling

If the parser encounters syntax it cannot understand, it produces `Junk` entries rather than failing entirely:

```ebnf
Junk      ::= junk_line (junk_line - "#" - "-" - [a-zA-Z])*
junk_line ::= /[^\n]*/ ("\u000A" | EOF)
```

Junk continues until the parser finds a line starting with `#`, `-`, or a letter (which could be a new entry). This means a syntax error in one message does NOT break parsing of subsequent messages — a critical robustness feature.

Source: EBNF spec

### 5. Identifier Restrictions

Message identifiers MUST start with a letter `[a-zA-Z]` and can only contain letters, digits, underscores, and hyphens. NO dots, spaces, or other special characters:

```ebnf
Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*
```

Term identifiers follow the same rule but are prefixed with `-` at the syntax level (the `-` is NOT part of the Identifier production).

### 6. Named Argument Values Are Restricted

In function calls and term references, named argument values MUST be string literals or number literals:

```ebnf
NamedArgument ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
```

You CANNOT pass a variable reference as a named argument value. This is NOT valid: `{ -term(case: $myCase) }`. The value must be a literal like `{ -term(case: "nominative") }`.

### 7. Default Variant is Mandatory

Every select expression MUST have exactly one default variant (marked with `*`). The grammar enforces this:

```ebnf
variant_list ::= Variant* DefaultVariant Variant* line_end
```

Omitting the default variant is a syntax error.

### 8. Tab Characters

Tabs (U+0009) are NOT whitespace for indentation purposes. A tab at the start of a continuation line will be treated as text content, not as indent. ALWAYS use spaces (U+0020) for indentation.

### 9. Comment Space Requirement

The comment syntax requires a space after the hash characters:

```ebnf
CommentLine ::= ("###" | "##" | "#") ("\u0020" comment_char*)? line_end
```

`#comment` is NOT valid. It must be `# comment` (with a space). An empty comment (`#` alone on a line) IS valid.

### 10. Message vs. Term Value Requirements

A Message can exist with ONLY attributes and no value. A Term MUST have a value:

```ebnf
Message ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
Term    ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
```

This is because terms are designed to be referenced in placeables, which requires a value to interpolate.

---

## Appendix: Complete EBNF Grammar (Fluent Syntax 1.0)

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
