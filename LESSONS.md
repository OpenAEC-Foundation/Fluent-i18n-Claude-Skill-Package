# Lessons Learned

Observations and findings captured during skill package development.

---

## L-001: FTL Whitespace Sensitivity is the Primary Pitfall

- **Date**: 2026-03-19
- **Context**: Project Fluent uses indentation to determine multiline message boundaries.
- **Finding**: FTL is whitespace-sensitive — continuation lines MUST be indented relative to the message identifier. This is the single most common source of errors when AI models generate FTL. Skills MUST emphasize indentation rules prominently. Unlike YAML or Python, FTL uses indentation to delimit the end of a message value, not braces or quotes.

---

## L-002: Fluent is NOT ICU MessageFormat

- **Date**: 2026-03-19
- **Context**: Many i18n libraries use ICU MessageFormat syntax. Fluent is a distinct format.
- **Finding**: Claude frequently confuses Fluent FTL syntax with ICU MessageFormat when generating localization code. Key differences: Fluent uses `{ $var }` not `{var}`, select expressions use `->` not `select`, plural categories are variant keys not keywords. Skills MUST explicitly call out these differences to prevent cross-contamination from Claude's training data.

---

## L-003: Terms vs Messages Have Different Visibility Rules

- **Date**: 2026-03-19
- **Context**: FTL has both messages (public) and terms (private/internal).
- **Finding**: Terms (prefixed with `-`) are NEVER exposed through the Fluent API — `getMessage("-brand-name")` returns undefined. Terms exist solely for reuse within other messages. This distinction is critical and frequently misunderstood. Skills covering terms MUST make this visibility rule prominent.
