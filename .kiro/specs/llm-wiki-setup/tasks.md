# Implementation Plan: LLM Wiki Setup

## Overview

All tasks create files or directories. There is no application code, no build step, and no tests. Every task is independent — files do not depend on each other — so most can run in parallel.

## Tasks

- [x] 1. Scaffold the directory structure
  - Create `raw/` at workspace root (skip if already exists)
  - Create `raw/assets/` subdirectory (skip if already exists)
  - Create `wiki/` at workspace root (skip if already exists)
  - Create `wiki/sources/` subdirectory (skip if already exists)
  - Create `wiki/entities/` subdirectory (skip if already exists)
  - Create `wiki/concepts/` subdirectory (skip if already exists)
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.9_

- [x] 2. Create wiki/index.md
  - [x] 2.1 Create `wiki/index.md` with initial content
    - Top-level heading `# Wiki Index`
    - 1–3 sentence description of purpose
    - Three sections: `## Sources`, `## Entities`, `## Concepts`
    - Each section contains `_No pages yet._` as placeholder
    - Skip if file already exists
    - _Requirements: 1.7, 8.1, 8.2, 8.3_

- [x] 3. Create wiki/log.md
  - [x] 3.1 Create `wiki/log.md` with initial content
    - Top-level heading `# Wiki Log`
    - Description of append-only purpose (≤3 sentences)
    - Fenced code block showing the Log_Entry format: `## [YYYY-MM-DD] <operation> | <title>`
    - Skip if file already exists
    - _Requirements: 1.8, 9.1, 9.2_

- [x] 4. Create .kiro/steering/wiki-schema.md
  - [x] 4.1 Create `.kiro/steering/wiki-schema.md`
    - Frontmatter: `inclusion: always`
    - Section 1 — Three-Layer Architecture: name each layer (`raw/`, `wiki/`, Schema), state purpose, state agent read/write access per layer
    - Section 2 — Immutability of `raw/`: label `raw/` as immutable; enumerate all five prohibited operations: write, modify, move, rename, delete; applies to all files under `raw/` and subdirectories
    - Section 3 — Wiki Page Types and Required Frontmatter: source summary pages require `type`, `source`, `date`; entity pages require `type`, `aliases`; concept pages require `type`, `related`
    - Section 4 — Linking Convention: standard markdown links `[Page Title](../relative/path.md)` for all cross-references
    - Section 5 — Citation Format: `[filename](../../raw/filename)` referencing actual file path
    - Section 6 — Log Entry Format: `## [YYYY-MM-DD] <operation> | <title>`
    - Section 7 — Hard Rules: agent SHALL never modify `raw/`; agent SHALL never invent citations; agent SHALL never delete a wiki page without explicit user confirmation
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9, 10.1_

- [x] 5. Create .kiro/steering/ingest-workflow.md
  - [x] 5.1 Create `.kiro/steering/ingest-workflow.md`
    - Frontmatter: `inclusion: manual`
    - Activation section: tag `#ingest-workflow`, instructions for how to activate
    - Ingest sequence (7 ordered steps): read source, discuss takeaways, create source summary page in `wiki/sources/` with required frontmatter, update `wiki/index.md`, update 5–15 entity/concept pages (all applicable when fewer than 5 exist), flag contradictions, append Log_Entry to `wiki/log.md`
    - Page update budget section: 5–15 pages on populated wiki; all applicable pages when fewer than 5 exist
    - Contradiction handling section: mark affected pages with `> **Contradiction:**` callout containing existing claim and contradicting claim; do not overwrite silently
    - Log entry section: final step appends `## [YYYY-MM-DD] ingest | <source filename>`
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

- [x] 6. Create .kiro/steering/query-workflow.md
  - [x] 6.1 Create `.kiro/steering/query-workflow.md`
    - Frontmatter: `inclusion: manual`
    - Activation section: tag `#query-workflow`
    - Query sequence (7 ordered steps): read `wiki/index.md` (halt if unreadable), identify up to 10 relevant pages, read those pages, compose answer with ≥1 citation, present to user, prompt user to confirm filing, only if confirmed: save to `wiki/concepts/`, update `wiki/index.md`, append Log_Entry
    - Index unavailable section: notify user and halt without modifying any wiki files
    - Filing an answer section: location `wiki/concepts/<slug>.md`, required frontmatter `type: concept` + `related`, update `wiki/index.md` under `## Concepts`, append `## [YYYY-MM-DD] query | <page title>`
    - Log entry section (only when answer is filed)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

- [x] 7. Create .kiro/steering/lint-workflow.md
  - [x] 7.1 Create `.kiro/steering/lint-workflow.md`
    - Frontmatter: `inclusion: manual`
    - Activation section: tag `#lint-workflow`
    - Lint sequence: five ordered checks — contradictions, stale claims, orphan pages, missing cross-references, data gaps
    - Index unavailable handling: report as blocking issue, stop before all other checks
    - Issue handling: present consolidated list of all issues; wait for explicit user confirmation before modifying any wiki pages
    - Log entry section: append `## [YYYY-MM-DD] lint | <N> issues found` upon completion (whether finished or terminated early)
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

- [x] 8. Create hook files
  - [x] 8.1 Create `.kiro/hooks/ingest-on-new-source.kiro.hook`
    - Event type: `fileCreated`
    - File pattern: `raw/**/*`
    - Action: `askAgent`
    - Prompt instructs agent to activate `#ingest-workflow` and ingest the newly created file at `{{filePath}}`
    - _Requirements: 6.1, 6.2, 6.3, 6.4_
  - [x] 8.2 Create `.kiro/hooks/lint-wiki.kiro.hook`
    - Event type: `userTriggered`
    - Action: `askAgent`
    - Prompt instructs agent to activate `#lint-workflow` and run all five lint checks
    - _Requirements: 7.1, 7.2, 7.3_

- [ ] 9. Update wiki-schema.md — extended frontmatter and page structure
  - [ ] 9.1 Update Section 3 (Wiki Page Types and Required Frontmatter) in `.kiro/steering/wiki-schema.md`
    - Entity pages: add `created` (system-set, never LLM-generated), `sources` (array of wiki-links), `tags` (entity subtype); mark `aliases` and `reviewed` as optional
    - Concept pages: add `created` (system-set, never LLM-generated), `sources` (array of wiki-links), `tags` (concept subtype); mark `aliases` and `reviewed` as optional
    - Source summary pages: replace `source`/`date` with `sources` (array of wiki-links to pages created from this source), `tags`, `created`, `updated` (both system-set)
    - Add note: `created` and `updated` are set by the system — the Kiro_Agent SHALL leave them as empty strings or placeholders
    - Add note: on merge, `created` is preserved (older value kept); `updated` is always set to the current date by the system
    - Add `reviewed` flag definition: when `reviewed: true`, agent may only append genuinely new information
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5, 11.6_
  - [ ] 9.2 Add Section 8 (Required Page Section Structure) to `.kiro/steering/wiki-schema.md`
    - Entity page sections in order: Description, Related Entities, Related Concepts, Mentions in Source
    - Concept page sections in order: Definition, Key Characteristics, Applications, Related Concepts, Related Entities, Mentions in Source
    - Source summary page sections in order: Summary, Key Points, Mentioned Pages
    - Rule: all required sections must be present in order; sections may be empty but must not be omitted
    - _Requirements: 12.1, 12.2, 12.3, 12.4_
  - [ ] 9.3 Add Section 9 (Mentions in Source Format) to `.kiro/steering/wiki-schema.md`
    - Format: `"Verbatim quote in original language (optional translation)" — [[source-name|display-name]]`
    - Rule: quotes must be verbatim — paraphrase and summarization are prohibited in this section
    - Rule: each quote must include a source wiki-link for merge traceability
    - Rule: multiple quotes from the same source go in one block, separated by newlines
    - Rule: quote language is always the original source language; optional translation in parentheses
    - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5_
  - [ ] 9.4 Update Section 4 (Linking Convention) in `.kiro/steering/wiki-schema.md`
    - Replace relative markdown link convention with wiki-link syntax: `[[entities/page-name|Display Name]]` / `[[concepts/page-name|Display Name]]`
    - Specify path format: full path from `wiki/` root, no leading slash
    - Distinguish wiki-links (internal cross-references) from markdown links (citations to `raw/` files only)
    - Specify where each syntax applies: wiki-links in Related Entities, Related Concepts, Mentions in Source, and frontmatter arrays; markdown links in citations only
    - _Requirements: 15.1, 15.2, 15.3, 15.4_
  - [ ] 9.5 Add Section 10 (Naming Conventions) to `.kiro/steering/wiki-schema.md`
    - Filenames: lowercase-with-hyphens (slug) format
    - Entity/concept names: preserve original source language, never translate
    - Wiki-link display names: must match the canonical `# Heading` of the target page
    - _Requirements: 16.1, 16.2, 16.3_
  - [ ] 9.6 Add Section 11 (Maintenance Policies) to `.kiro/steering/wiki-schema.md`
    - Stale threshold: 90 days without updates
    - Contradiction severity levels: warning, conflict, error
    - Orphan page definition: no inbound links from other wiki pages
    - Missing page definition: referenced by `[[link]]` but does not exist
    - _Requirements: 17.3_

- [ ] 10. Update ingest-workflow.md — multi-source merge rules
  - [ ] 10.1 Add Multi-Source Merge Rules section to `.kiro/steering/ingest-workflow.md`
    - `sources` array: append new source wiki-link; never overwrite existing entries
    - `aliases`: append new alternative names; never remove existing aliases
    - `reviewed: true` pages: only append genuinely new information; do not alter, rewrite, or replace existing content
    - `NO_NEW_CONTENT`: if source adds nothing new to a page, skip the update and note `NO_NEW_CONTENT` in working summary
    - Contradictions: record in a dedicated `## Contradictions` section on the affected page (in addition to inline callout), with both claims and source attribution
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5_

- [ ] 11. Update lint-workflow.md — stale threshold and orphan definition
  - [ ] 11.1 Update Check 2 (Stale Claims) in `.kiro/steering/lint-workflow.md`
    - Define stale threshold as 90 days
    - Use `date` frontmatter on source summary pages to compare recency
    - _Requirements: 17.1, 17.2_
  - [ ] 11.2 Update Check 3 (Orphan Pages) in `.kiro/steering/lint-workflow.md`
    - Redefine orphan as: a page with no inbound links from any other wiki page (scan all wiki pages for wiki-links and markdown links)
    - Treat absence from `wiki/index.md` as a separate index drift issue, not the orphan definition
    - _Requirements: 18.1, 18.2, 18.3_

## Notes

- No tests, build steps, or application code are involved — this is a pure configuration feature
- All tasks are idempotent: skip creation if the target path already exists
- Tasks 2–8 are fully independent and can run in parallel after task 1 completes
- Tasks 9–11 are updates to existing steering files and can run in parallel with each other
- Hook files (8.1, 8.2) can be created in parallel; neither depends on the steering files

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["2.1", "3.1", "4.1", "5.1", "6.1", "7.1", "8.1", "8.2"] },
    { "id": 2, "tasks": ["9.1", "9.2", "9.3", "9.4", "9.5", "9.6", "10.1", "11.1", "11.2"] }
  ]
}
```
