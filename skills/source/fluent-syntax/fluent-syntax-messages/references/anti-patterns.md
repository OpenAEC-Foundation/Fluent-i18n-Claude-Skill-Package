# Anti-Patterns — FTL Message Syntax

Common mistakes when writing FTL messages, with explanations and corrections.

---

## 1. Using Tabs for Indentation

**Severity: CRITICAL** — This is the most common AI-generated FTL mistake.

Tabs (U+0009) are NOT whitespace in FTL. They are treated as regular text characters. The EBNF defines `blank_inline ::= "\u0020"+` — only space characters count.

```ftl
# WRONG — tabs used for indentation (shown as →)
message =
→   This tab becomes part of the output text.
→   The message is NOT properly indented.

# CORRECT — spaces used for indentation
message =
    This is properly indented with spaces.
    Continuation lines use spaces only.
```

**Why it breaks**: A tab at the start of a line is treated as text content, not as indentation. The parser may not recognize the line as a continuation, or the tab character will appear in the output string.

**Rule**: ALWAYS configure your editor to use spaces (not tabs) for `.ftl` files. NEVER rely on auto-indent that might insert tabs.

---

## 2. Missing Space After Comment Hash

```ftl
# WRONG — no space after #
#This is not a valid comment
##Section without space
###File comment without space

# CORRECT — space required after hash(es)
# This is a valid comment
## Section comment
### File comment

# ALSO CORRECT — empty comment (hash alone)
#
```

**Why it breaks**: The EBNF grammar requires `("###" | "##" | "#") ("\u0020" comment_char*)?` — the space (U+0020) is mandatory before comment text. Without it, the parser produces a Junk entry.

---

## 3. Special Characters at Continuation Line Start

```ftl
# WRONG — [ at line start is parsed as variant key
error-list =
    The following errors occurred:
    [Error 1] File not found
    [Error 2] Permission denied

# CORRECT — escape with quoted text placeable
error-list =
    The following errors occurred:
    {"["}Error 1] File not found
    {"["}Error 2] Permission denied
```

```ftl
# WRONG — * at line start is parsed as default variant marker
footnote =
    Important information below
    * First footnote
    * Second footnote

# CORRECT
footnote =
    Important information below
    {"*"} First footnote
    {"*"} Second footnote
```

```ftl
# WRONG — . at line start is parsed as attribute accessor
range =
    Enter a value between 1
    .0 and 10.0

# CORRECT
range =
    Enter a value between 1
    {"."}0 and 10.0
```

**Why it breaks**: The EBNF defines `indented_char ::= text_char - "[" - "*" - "."`. These three characters are reserved syntax at line start and trigger different parsing paths.

---

## 4. Unescaped Curly Braces

```ftl
# WRONG — literal { starts a placeable parse
json-example = The format is {key: value}

# CORRECT — escape both braces
json-example = The format is {"{"} key: value {"}"}
```

**Why it breaks**: `{` always opens a placeable expression. The parser attempts to parse `key: value}` as an expression, which fails and produces Junk.

---

## 5. Expecting Leading Whitespace to Be Preserved

```ftl
# WRONG — expecting 4 leading spaces in output
indented-code =     function hello() {

# The output is: "function hello() {"
# Leading spaces are stripped by dedentation.

# CORRECT — use quoted text placeable
indented-code = {"    "}function hello() {"{"}
```

**Why it breaks**: The dedentation algorithm strips common indent. Leading spaces on the first line (same line as `=`) are part of the indent calculation and get removed.

---

## 6. Blank Line Between Comment and Message

```ftl
# WRONG if you want the comment bound to the message
# This describes the greeting message

greeting = Hello!
# The blank line above breaks the binding.
# Localization tools will NOT show this comment with 'greeting'.

# CORRECT — comment directly above, no blank line
# This describes the greeting message
greeting = Hello!
```

**Why it breaks**: A `#` comment binds to the next message ONLY when there is no blank line between them. A blank line makes the comment standalone.

---

## 7. Using Identifiers with Invalid Characters

```ftl
# WRONG — dots are not allowed in identifiers
user.name = User Name

# WRONG — spaces are not allowed
user name = User Name

# WRONG — starts with a number
1st-place = First Place

# WRONG — starts with underscore
_private = Private

# CORRECT
user-name = User Name
first-place = First Place
private-msg = Private
```

**Why it breaks**: `Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*` — must start with a letter, and only letters, digits, underscores, and hyphens are allowed.

---

## 8. Expecting Escape Sequences in Regular Text

```ftl
# WRONG — \n is NOT a newline in regular text
message = First line\nSecond line
# Output: "First line\nSecond line" (literal backslash-n)

# CORRECT — use multiline text for line breaks
message =
    First line
    Second line

# CORRECT — use unicode escape in quoted text for special chars
message = First line{"\u000A"}Second line
```

**Why it breaks**: Escape sequences (`\n`, `\t`, `\uXXXX`) work ONLY inside quoted text (string literals within `""`). In regular FTL text, the backslash is just a regular character.

---

## 9. Forgetting the `=` Sign

```ftl
# WRONG — missing equals sign
greeting Hello, world!

# CORRECT
greeting = Hello, world!
```

**Why it breaks**: Without `=`, the parser cannot recognize the line as a message. It becomes Junk. The message syntax requires `Identifier blank_inline? "=" blank_inline? Pattern`.

---

## 10. Mixing Up Variable and Message References

```ftl
# WRONG — using $ for message references
help = Click { $menu-save } to save.
# This looks for a runtime variable, not the menu-save message.

# CORRECT — no $ for message references
menu-save = Save
help = Click { menu-save } to save.

# WRONG — omitting $ for variables
welcome = Welcome, { user }!
# This references a message called "user", not a runtime variable.

# CORRECT — $ prefix for variables
welcome = Welcome, { $user }!
```

**Why it breaks**: `$name` is a `VariableReference` (runtime data from the app). `name` without `$` is a `MessageReference` (another FTL message). Mixing them up causes either missing values or resolution errors.
