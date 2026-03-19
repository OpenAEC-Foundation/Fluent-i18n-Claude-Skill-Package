# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0] - 2026-03-19

### Added
- 16 deterministic skills for Project Fluent (Mozilla i18n)
- **fluent-core/** (3 skills): architecture, bundle-api, langneg
- **fluent-syntax/** (5 skills): messages, selectors, terms, attributes, functions
- **fluent-impl/** (4 skills): react, locale-loading, locale-switching, project-setup
- **fluent-errors/** (2 skills): parsing, resolution
- **fluent-agents/** (2 skills): review, project-scaffolder
- Deep research document (vooronderzoek-fluent.md) covering FTL spec, @fluent/bundle, @fluent/react, @fluent/langneg
- Definitive masterplan with 16 skills across 6 batches
- INDEX.md with complete skill catalog
- 11 architectural decisions (D-001 through D-011)
- 6 lessons learned (L-001 through L-006)
- Approved sources registry with 15+ official URLs
- All skills verified against official Fluent specification and npm packages

### Technical Details
- All SKILL.md files under 500 lines (range: 199-418)
- Each skill has 3 reference files (methods.md, examples.md, anti-patterns.md)
- English-only content with ALWAYS/NEVER deterministic language
- Dual FTL + TypeScript coverage for integration skills
- 25+ consolidated anti-patterns across all skills
- Verified against: Fluent Syntax 1.0, @fluent/bundle 0.19.1, @fluent/react 0.15.2, @fluent/langneg 0.7.0

## [0.1.0] - 2026-03-19

### Added
- Project initialized with 7-phase research-first methodology
- Core documentation files (CLAUDE.md, ROADMAP.md, DECISIONS.md, etc.)
- Directory structure for skills and research
- Approved sources registry with Project Fluent official documentation
