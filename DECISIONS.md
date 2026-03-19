# DECISIONS

Architectural and process decisions with rationale. Each decision is numbered and immutable once recorded. New decisions may supersede old ones but old ones are never deleted.

---

## D-001: 7-Phase Research-First Methodology
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need a structured approach to build high-quality skills
**Decision**: Adopt the 7-phase methodology proven in the ERPNext, Blender-Bonsai, and Tauri Skill Packages
**Rationale**: ERPNext successfully produced 28 domain skills, Blender-Bonsai produced 73 skills, Tauri produced 27 skills with this approach. Research-first prevents hallucinated content.
**Reference**: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package/blob/main/WAY_OF_WORK.md

## D-002: Single Technology Package
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Project Fluent is one technology (FTL syntax + JavaScript runtime libraries)
**Decision**: No per-technology separation needed. All skills share the `fluent-` prefix under a single `skills/source/` tree.
**Rationale**: Project Fluent is a single localization system. The FTL files and TypeScript integration code are two sides of the same framework, not separate technologies.

## D-003: English-Only Skills
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Team works primarily in Dutch, skills target international audience
**Decision**: ALL skill content in English only
**Rationale**: Skills are instructions for Claude, not end-user documentation. Claude reads English and responds in any language. Bilingual skills double maintenance with zero functional benefit. Proven in ERPNext, Blender, and Tauri projects.

## D-004: Claude Code Agent Tool for Orchestration
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need to produce skills efficiently. Windows environment, no oa-cli available.
**Decision**: Use Claude Code Agent tool for parallel execution instead of oa-cli
**Rationale**: Windows environment does not support oa-cli (requires WSL/Linux). Claude Code Agent tool provides native parallelism within the Claude Code session.

## D-005: MIT License
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need to choose open-source license
**Decision**: MIT License
**Rationale**: Most permissive, maximizes adoption. Consistent with OpenAEC Foundation philosophy.

## D-006: Fluent Specification as Primary Authority
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Project Fluent has a formal specification and multiple implementations
**Decision**: The FTL specification at projectfluent.org is the primary authority for all syntax-related skills. The @fluent/bundle and @fluent/react npm packages are the primary authority for API-related skills.
**Rationale**: The specification defines what is correct FTL. Implementations may have bugs or extensions, but the spec is the contract. For API skills, the JavaScript packages are the target platform per project scope.

## D-007: Dual Coverage — FTL + TypeScript
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Fluent localization involves FTL files (translations) and TypeScript code (runtime integration)
**Decision**: Every integration-related skill MUST show both FTL files and TypeScript integration code
**Rationale**: A skill that shows only the FTL syntax without the TypeScript code to load and format it (or vice versa) is incomplete. Developers need to see both sides to implement localization correctly.

## D-008: SKILL.md < 500 Lines
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Skills must be concise enough to fit in Claude's context window
**Decision**: SKILL.md MUST be under 500 lines. Heavy content (method tables, extensive examples, anti-pattern catalogs) goes in references/ subdirectory files.
**Rationale**: Proven constraint from ERPNext, Blender, and Tauri packages. Forces prioritization of the most critical information in the main skill file.

## D-009: WebFetch for Source Verification
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: AI training data may contain outdated or incorrect API information
**Decision**: All API claims and code patterns MUST be verified via WebFetch against official documentation during research phases.
**Rationale**: Prevents hallucinated APIs and outdated patterns. Ensures skills reflect the actual current state of the Fluent ecosystem.

## D-010: Phase 4 Skipped — Covered by Phase 2
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Phase 4 (topic-specific research) was planned as per-skill research
**Decision**: Skip Phase 4 because Phase 2 deep research already covers all topics with sufficient depth for skill creation.
**Rationale**: The vooronderzoek document (3 parts, ~7000 words) covers FTL syntax, @fluent/bundle API, and @fluent/react+langneg with complete code examples, method signatures, and anti-patterns. Additional per-skill research would be redundant.

## D-011: Masterplan Refinement — 3 Merges
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Phase 3 refinement of 19 raw skills against research findings
**Decision**: Merge 3 skills: syntax-advanced→split into messages+functions, impl-dom-overlays→impl-react, errors-debugging→errors-resolution. Result: 16 definitive skills.
**Rationale**: "Advanced" is too vague a category. DOM overlays are part of the React component API. Debugging is a cross-cutting concern covered in error resolution.
