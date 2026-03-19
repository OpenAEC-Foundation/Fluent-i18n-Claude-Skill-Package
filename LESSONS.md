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

---

## L-004: Phase 4 (Topic Research) Can Be Skipped When Phase 2 Is Thorough

- **Date**: 2026-03-19
- **Context**: The 7-phase methodology includes Phase 4 for per-skill topic research.
- **Finding**: When Phase 2 deep research is comprehensive enough (3 parallel agents covering spec, API, and React/langneg — ~7000 words total), Phase 4 becomes redundant. For a single-technology package like Fluent, the vooronderzoek covers all topics. This saved significant time. For multi-technology packages (like Blender-Bonsai), Phase 4 is still needed.

---

## L-005: 16 Skills Is the Sweet Spot for a Single-Technology Package

- **Date**: 2026-03-19
- **Context**: Raw masterplan had 19 skills, refined to 16 after research.
- **Finding**: Three merges (advanced→split, dom-overlays→react, debugging→resolution) improved the package by eliminating vague categories and thin standalone skills. For a single-technology package, 15-20 skills appears optimal — enough to be comprehensive, few enough to maintain quality. Compare: ERPNext (28, multi-domain), Tauri (27, multi-layer), Blender (73, multi-technology).

---

## L-006: Agent Rate Limits Are Non-Fatal When Files Are Written Eagerly

- **Date**: 2026-03-19
- **Context**: One agent hit a rate limit during Batch 4.
- **Finding**: Because agents write files as they go (not at the end), a rate limit after file creation means the skill is complete even though the agent reports failure. Always check if files exist before re-running a failed agent.
