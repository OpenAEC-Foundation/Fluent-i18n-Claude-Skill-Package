# Examples — FTL Message Syntax

All examples verified against Fluent Syntax 1.0 specification and projectfluent.org guides.

---

## Simple Messages

```ftl
hello = Hello, world!
app-title = My Application
logout = Log out
sign-in = Sign in to your account
```

The identifier is the key used by developers in code. The value begins at the first non-blank character after `=`.

---

## Multiline Messages — Inline Start

Text continues on the next line when indented by at least one space:

```ftl
about-app = This application helps you manage
    your daily tasks and stay organized
    throughout the week.
```

Output: `This application helps you manage your daily tasks and stay organized throughout the week.`

Line breaks in the source become spaces in the output. Each continuation line MUST be indented by at least one space.

---

## Multiline Messages — Block Format

Starting the value on the line after `=` is called "block" format:

```ftl
about-app =
    This application helps you manage
    your daily tasks and stay organized
    throughout the week.
```

Output is identical to the inline start example. Block format is often more readable for longer messages.

---

## Dedentation in Action

Common indent is stripped from all indented lines:

```ftl
indented-example =
    Line one (4 spaces of indent in source).
        Line two (8 spaces in source, 4 preserved).
    Line three (4 spaces in source, 0 preserved).
```

Output:
```
Line one (4 spaces of indent in source).
    Line two (8 spaces in source, 4 preserved).
Line three (4 spaces in source, 0 preserved).
```

The common indent (4 spaces) is removed. Line two's extra 4 spaces are preserved.

---

## Blank Line Handling

```ftl
# Leading blank lines are ignored
leading =


    This value starts with "This".

# Trailing blank lines are ignored
trailing =
    This value ends with "end".


# Interior blank lines are preserved
interior =
    First paragraph.

    Second paragraph after a blank line.
```

Output for `interior`: `First paragraph.\n\nSecond paragraph after a blank line.`

---

## Variable References

```ftl
welcome = Welcome, { $user }!
unread-emails = { $user } has { $email-count } unread emails.
time-elapsed = Time elapsed: { $duration }s.
download-progress = Downloaded { $current } of { $total } files.
```

Variables use the `$` prefix. Values are provided by the application at runtime. Numbers and dates are automatically formatted per the user's locale.

---

## Message References

```ftl
-brand-name = Acme Corp

menu-save = Save
menu-open = Open
help-menu-save = Click { menu-save } to save the file.
help-menu-open = Click { menu-open } to open a file.
```

Message references use the identifier without `$`. The referenced message's value is interpolated into the current message.

---

## Comments — All Three Levels

```ftl
### Localization for the Settings page of MyApp
### This file contains all user-facing strings for settings.

## Account Settings

# Shown at the top of the account settings panel
account-heading = Account Settings

# The user's display name field label
account-name = Display Name

## Notification Settings

# Toggle for enabling push notifications
notifications-toggle = Enable Notifications

# $count (Number) - The number of unread notifications
notifications-badge = { $count } unread
```

### Comment Binding Example

```ftl
# This comment IS bound to the message below (no blank line)
bound-message = This message has a comment.

# This comment is NOT bound to the message below (blank line separates)

unbound-message = This message has no bound comment.
```

---

## Special Characters — Curly Braces

```ftl
json-hint = Use {"{"}"key": "value"{"}"} format for JSON objects.
css-rule = The selector {"{"} color: red; {"}"} applies globally.
```

Output for `json-hint`: `Use {"key": "value"} format for JSON objects.`

---

## Special Characters — Restricted at Line Start

```ftl
# Using [ at line start
list-example =
    Items in this list:
    {"["}1] First item
    {"["}2] Second item

# Using * at line start
footnote =
    Important information
    {"*"} This is a footnote marker

# Using . at line start
ellipsis =
    Loading
    {"."}.. please wait
```

---

## Preserving Leading Whitespace

```ftl
# Without placeable: leading spaces are stripped
stripped = This starts at "This" despite spaces in source.

# With quoted text placeable: leading spaces are preserved
code-indent = {"    "}if (condition) {"{"}
    {"        "}doSomething();
    {"    "}
```

---

## Unicode Escape Sequences (In Quoted Text Only)

```ftl
# Non-breaking space between words
privacy-label = Privacy{"\u00A0"}Policy

# Em dash
long-dash = First part{"\u2014"}second part

# Backslash in regular text is literal (no escaping)
file-path = C:\Users\Documents\file.txt
```

---

## Complete Real-World Example

```ftl
### Localization for the Email Client — Inbox view

## Toolbar actions

inbox-title = Inbox
compose-new = Compose

# $count (Number) - Total number of unread emails
unread-count = { $count } unread

## Email list

# $sender (String) - The name of the email sender
# $subject (String) - The email subject line
email-item = { $sender } — { $subject }

# Shown when the inbox is empty
empty-inbox =
    Your inbox is empty.

    New messages will appear here when they arrive.

## Status bar

# $synced (Number) - Number of synced emails
# $total (Number) - Total number of emails
sync-status = Synced { $synced } of { $total } emails.
last-sync = Last synced: { $time }.
```
