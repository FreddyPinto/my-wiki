# Requirements Document

## Introduction

This feature sets up the LLM Wiki pattern in a Kiro workspace. The LLM Wiki pattern replaces one-shot RAG with a persistent, incrementally maintained knowledge base — a structured directory of markdown files that an LLM builds and updates over time. Instead of re-deriving knowledge on every query, the LLM compounds knowledge across sessions.

The workspace has three layers: immutable raw sources that the LLM reads but never modifies, a LLM-owned wiki of structured markdown pages, and a schema encoded as Kiro steering files that governs how the wiki is organized and how the LLM should behave. Three operations — Ingest, Query, and Lint — drive the workflow.

## Glossary

- **Wiki**: The `wiki/` directory containing all LLM-generated and LLM-maintained markdown files.
- **Raw Sources**: The `raw/` directory containing immutable source documents (articles, papers, images, data) that the LLM reads but never modifies.
- **Schema**: The set of Kiro steering files that define wiki structure, conventions, and operational workflows.
- **Ingest**: The operation of reading a new source, extracting key information, and integrating it into the wiki.
- **Query**: The operation of answering a user question by reading the wiki and optionally filing the answer as a new wiki page.
- **Lint**: The operation of performing a health check on the wiki to find contradictions, stale claims, orphan pages, and missing cross-references.
- **Index**: The `wiki/index.md` file — a content-oriented catalog of all pages in the wiki.
- **Log**: The `wiki/log.md` file — an append-only chronological record of all wiki operations.
- **Steering File**: A markdown file in `.kiro/steering/` that provides always-on or manually-activated instructions to the Kiro agent.
- **Hook**: A Kiro automation in `.kiro/hooks/` that triggers an agent action based on an IDE event.
- **Entity Page**: A wiki page in `wiki/entities/` describing a specific named entity (person, organization, product, etc.).
- **Concept Page**: A wiki page in `wiki/concepts/` describing an abstract concept or topic.
- **Source Summary Page**: A wiki page in `wiki/sources/` summarizing a specific raw source document.
- **Frontmatter**: YAML metadata block at the top of a wiki markdown page.
- **Kiro_Agent**: The Kiro AI agent that executes ingest, query, and lint operations.
- **Log_Entry**: A single record appended to `wiki/log.md` following the format `## [YYYY-MM-DD] <operation> | <title>`.

---

## Requirements

### Requirement 1: Directory Structure Initialization

**User Story:** As a knowledge worker, I want the wiki directory structure scaffolded in my workspace, so that the LLM Wiki pattern has a consistent layout to operate against.

#### Acceptance Criteria

1. THE Workspace SHALL contain a `raw/` directory at the root intended for source documents that the Kiro_Agent reads but does not modify.
2. THE Workspace SHALL contain a `raw/assets/` subdirectory for binary and media source files.
3. THE Workspace SHALL contain a `wiki/` directory at the root for all LLM-generated content.
4. THE Workspace SHALL contain a `wiki/sources/` subdirectory for source summary pages.
5. THE Workspace SHALL contain a `wiki/entities/` subdirectory for entity pages.
6. THE Workspace SHALL contain a `wiki/concepts/` subdirectory for concept pages.
7. THE Workspace SHALL contain a `wiki/index.md` file initialized with a top-level heading (`# Wiki Index`) and a placeholder entry in each section indicating the wiki is empty.
8. THE Workspace SHALL contain a `wiki/log.md` file initialized with a top-level heading (`# Wiki Log`) and a fenced code block showing the Log_Entry format.
9. WHEN the workspace initialization is run more than once, THE initialization SHALL NOT overwrite or reset any `raw/`, `wiki/`, or subdirectory that already exists, preserving all existing files.

---

### Requirement 2: Wiki Schema Steering File

**User Story:** As a knowledge worker, I want an always-on steering file that defines the wiki's structure and hard rules, so that the Kiro agent consistently follows the correct conventions in every session.

#### Acceptance Criteria

1. THE Workspace SHALL contain a `.kiro/steering/wiki-schema.md` steering file with `inclusion: always` front matter.
2. THE wiki-schema.md SHALL describe the three-layer architecture by naming each layer (`raw/`, `wiki/`, and Schema), stating the purpose of each, and specifying which layers the Kiro_Agent may read versus write.
3. THE wiki-schema.md SHALL enumerate the required frontmatter fields for each wiki page type: source summary pages SHALL include `type`, `source`, and `date`; entity pages SHALL include `type` and `aliases`; concept pages SHALL include `type` and `related`.
4. THE wiki-schema.md SHALL define the linking convention as standard markdown links of the form `[Page Title](../relative/path.md)` used consistently for all cross-references between wiki pages.
5. THE wiki-schema.md SHALL define the citation format as a markdown link of the form `[filename](../../raw/filename)` referencing the actual file path under `raw/`.
6. THE wiki-schema.md SHALL define the Log_Entry format as `## [YYYY-MM-DD] <operation> | <title>`.
7. THE wiki-schema.md SHALL state the hard rule that the Kiro_Agent SHALL never modify any file under `raw/`.
8. THE wiki-schema.md SHALL state the hard rule that the Kiro_Agent SHALL never invent citations that do not correspond to a file in `raw/`.
9. THE wiki-schema.md SHALL state the hard rule that the Kiro_Agent SHALL never delete a wiki page without first presenting the page path to the user and receiving an explicit "yes" or equivalent confirmation in the same session.

---

### Requirement 3: Ingest Workflow Steering File

**User Story:** As a knowledge worker, I want a detailed ingest workflow steering file, so that the Kiro agent follows a consistent and thorough sequence when integrating a new source into the wiki.

#### Acceptance Criteria

1. THE Workspace SHALL contain a `.kiro/steering/ingest-workflow.md` steering file with `inclusion: manual` and the activation tag `#ingest-workflow`.
2. THE ingest-workflow.md SHALL define the ingest sequence: read the new source, discuss takeaways with the user, create a source summary page in `wiki/sources/`, update `wiki/index.md`, update all entity and concept pages that mention entities or concepts directly introduced or materially updated by the new source, flag any contradictions found, and append a Log_Entry to `wiki/log.md`. Where fewer than 5 applicable pages exist in the wiki, the Kiro_Agent SHALL update all applicable pages.
3. THE ingest-workflow.md SHALL specify that the Kiro_Agent SHALL update between 5 and 15 wiki pages per ingest operation on a wiki with sufficient existing content; where fewer than 5 applicable pages exist, the Kiro_Agent SHALL update all applicable pages without error.
4. WHEN a contradiction between a new source and existing wiki content is detected, THE ingest-workflow.md SHALL instruct the Kiro_Agent to mark the affected wiki pages with a `> **Contradiction:**` callout block containing both the existing claim and the contradicting claim, rather than silently overwriting the existing content.
5. THE ingest-workflow.md SHALL instruct the Kiro_Agent to append a Log_Entry to `wiki/log.md` as the final step of every ingest operation.

---

### Requirement 4: Query Workflow Steering File

**User Story:** As a knowledge worker, I want a query workflow steering file, so that the Kiro agent follows a consistent sequence when answering questions from the wiki.

#### Acceptance Criteria

1. THE Workspace SHALL contain a `.kiro/steering/query-workflow.md` steering file with `inclusion: manual` and the activation tag `#query-workflow`.
2. THE query-workflow.md SHALL define the query sequence: read `wiki/index.md` first, identify relevant pages (up to 10), read those pages, compose an answer with citations, and explicitly prompt the user to confirm before filing the answer as a new wiki page.
3. THE query-workflow.md SHALL instruct the Kiro_Agent to include at least one citation referencing the specific wiki pages used when composing an answer.
4. WHEN the Kiro_Agent files an answer as a new wiki page, THE query-workflow.md SHALL instruct the Kiro_Agent to save the page under `wiki/concepts/`, update `wiki/index.md`, and append a Log_Entry to `wiki/log.md`.
5. WHEN `wiki/index.md` cannot be read at the start of a query, THE query-workflow.md SHALL instruct the Kiro_Agent to notify the user and halt without modifying any wiki files.

---

### Requirement 5: Lint Workflow Steering File

**User Story:** As a knowledge worker, I want a lint workflow steering file, so that the Kiro agent can periodically perform a structured health check on the wiki.

#### Acceptance Criteria

1. THE Workspace SHALL contain a `.kiro/steering/lint-workflow.md` steering file with `inclusion: manual` and the activation tag `#lint-workflow`.
2. THE lint-workflow.md SHALL define the lint sequence to check for: contradictions between wiki pages, stale claims contradicted by a newer source in `raw/`, orphan pages not linked from `wiki/index.md`, missing cross-references between pages sharing entities or concepts, and data gaps where topics are mentioned but lack dedicated pages.
3. WHEN one or more lint issues are found, THE lint-workflow.md SHALL instruct the Kiro_Agent to present a consolidated list of all issues to the user and receive explicit confirmation before modifying any wiki pages.
4. THE lint-workflow.md SHALL instruct the Kiro_Agent to append a Log_Entry to `wiki/log.md` upon completing a lint operation (whether the run finished fully or was terminated early by the user), including the count of issues found.
5. WHEN `wiki/index.md` is missing or unreadable at the start of a lint operation, THE lint-workflow.md SHALL instruct the Kiro_Agent to report this to the user as a blocking issue before proceeding with any other checks.

---

### Requirement 6: Ingest Hook

**User Story:** As a knowledge worker, I want a Kiro hook that automatically triggers the ingest workflow when a new file is added to `raw/`, so that new sources are integrated into the wiki without requiring manual intervention.

#### Acceptance Criteria

1. THE Workspace SHALL contain a `.kiro/hooks/ingest-on-new-source.kiro.hook` hook file.
2. THE ingest-on-new-source hook SHALL use the `fileCreated` event type.
3. THE ingest-on-new-source hook SHALL watch the file pattern `raw/**/*`.
4. WHEN a new file is created under `raw/`, THE ingest-on-new-source hook SHALL trigger the Kiro_Agent to execute the ingest workflow by activating the `#ingest-workflow` steering file and passing the newly created file's path as the source to ingest.

---

### Requirement 7: Lint Hook

**User Story:** As a knowledge worker, I want a manually-triggered Kiro hook that runs the lint workflow on demand, so that I can perform wiki health checks whenever I choose.

#### Acceptance Criteria

1. THE Workspace SHALL contain a `.kiro/hooks/lint-wiki.kiro.hook` hook file.
2. THE lint-wiki hook SHALL use the `userTriggered` event type.
3. WHEN the lint-wiki hook is triggered by the user, THE lint-wiki hook SHALL instruct the Kiro_Agent to execute the full lint workflow by activating the `#lint-workflow` steering file, covering all five checks defined in Requirement 5: contradictions, stale claims, orphan pages, missing cross-references, and data gaps.

---

### Requirement 8: Wiki Index Initialization

**User Story:** As a knowledge worker, I want the `wiki/index.md` initialized with a clear structure, so that the Kiro agent has a valid starting point to append to on every ingest.

#### Acceptance Criteria

1. THE wiki/index.md SHALL be initialized with a top-level heading (`# Wiki Index`) and a description of its purpose in 1–3 sentences.
2. THE wiki/index.md SHALL contain three named sections — `## Sources`, `## Entities`, and `## Concepts` — so the Kiro_Agent knows where to catalog each page type.
3. WHEN the wiki contains no pages yet, each section in `wiki/index.md` SHALL contain the placeholder line `_No pages yet._` so the Kiro_Agent has a clear, machine-readable signal of the empty state.

---

### Requirement 9: Wiki Log Initialization

**User Story:** As a knowledge worker, I want the `wiki/log.md` initialized as an append-only log, so that the Kiro agent has a valid file to append Log_Entries to from the first operation.

#### Acceptance Criteria

1. THE wiki/log.md SHALL be initialized with a top-level heading (`# Wiki Log`) and a description of its append-only purpose in no more than three sentences.
2. THE wiki/log.md SHALL display the Log_Entry format as a fenced code block (e.g. ` ```## [YYYY-MM-DD] <operation> | <title>``` `) so the format is unambiguously visible in the file itself.
3. THE Kiro_Agent SHALL only append to `wiki/log.md` by adding content after the last existing line; THE Kiro_Agent SHALL never alter, reorder, or remove any character of an existing Log_Entry.

---

### Requirement 10: Raw Sources Immutability

**User Story:** As a knowledge worker, I want the raw sources directory to be treated as read-only by the Kiro agent, so that original source documents are never accidentally modified or deleted.

#### Acceptance Criteria

1. THE wiki-schema.md SHALL contain a dedicated section that labels `raw/` as immutable and explicitly lists the prohibited operations: write, modify, move, rename, and delete — for all files under `raw/` including all subdirectories.
2. WHILE operating in the wiki workspace, THE Kiro_Agent SHALL read files in `raw/` but SHALL never write to, modify, move, rename, or delete any file under `raw/` or any of its subdirectories.
3. IF the Kiro_Agent is instructed to perform any prohibited operation on a file under `raw/`, THEN THE Kiro_Agent SHALL refuse the instruction and respond with a message stating that `raw/` is an immutable layer and the requested operation is not permitted.

---

### Requirement 11: Extended Frontmatter Schema

**User Story:** As a knowledge worker, I want all wiki pages to carry a richer set of metadata fields, so that pages are traceable, classifiable, and maintainable by both the agent and external tooling.

#### Acceptance Criteria

1. THE wiki-schema.md SHALL specify the following required frontmatter fields for **entity pages**: `type` (value: `entity`), `created` (ISO date), `sources` (array of wiki-links to source summary pages), and `tags` (entity subtype label). `aliases` and `reviewed` SHALL be defined as optional fields.
2. THE wiki-schema.md SHALL specify the following required frontmatter fields for **concept pages**: `type` (value: `concept`), `created` (ISO date), `sources` (array of wiki-links to source summary pages), and `tags` (concept subtype label). `aliases` and `reviewed` SHALL be defined as optional fields.
3. THE wiki-schema.md SHALL specify the following required frontmatter fields for **source summary pages**: `type` (value: `source`), `tags` (inherited from the source note), `sources` (array of wiki-links to pages created from this source), `created` (ISO date), and `updated` (ISO date).
4. THE wiki-schema.md SHALL state that `created` and `updated` are set by the system and SHALL never be LLM-generated. The Kiro_Agent SHALL leave these fields as empty strings or placeholders and SHALL NOT populate them with dates it computes.
5. THE wiki-schema.md SHALL state that on merge, `created` is preserved (the older value is kept) and `updated` is always set to the current date by the system.
6. THE wiki-schema.md SHALL define the optional `reviewed` field: WHEN `reviewed: true` is set on a page, THE Kiro_Agent SHALL treat the page as human-verified and SHALL only append genuinely new information rather than modifying or replacing existing content.

---

### Requirement 12: Required Page Section Structure

**User Story:** As a knowledge worker, I want every wiki page type to follow a consistent section structure, so that the agent produces predictable, well-organized pages that are easy to navigate and merge.

#### Acceptance Criteria

1. THE wiki-schema.md SHALL define the required section structure for **entity pages** as: (1) **Description** — 3–6 sentences with concrete facts and bidirectional links; (2) **Related Entities** — links to related entity pages; (3) **Related Concepts** — links to related concept pages; (4) **Mentions in Source** — verbatim quotes with source attribution.
2. THE wiki-schema.md SHALL define the required section structure for **concept pages** as: (1) **Definition** — clear, concise definition; (2) **Key Characteristics** — bullet list of defining traits; (3) **Applications** — real-world usage scenarios; (4) **Related Concepts** — links to related concept pages; (5) **Related Entities** — links to related entity pages; (6) **Mentions in Source** — verbatim quotes with source attribution.
3. THE wiki-schema.md SHALL define the required section structure for **source summary pages** as: (1) **Summary** — 2–4 sentence description of the source content; (2) **Key Points** — bullet list of main insights; (3) **Mentioned Pages** — list of entity and concept pages created from this source.
4. WHEN the Kiro_Agent creates or updates a wiki page, it SHALL ensure all required sections are present and in the defined order. Sections MAY be empty if no content applies, but they SHALL NOT be omitted.

---

### Requirement 13: Mentions in Source — Verbatim Quotes

**User Story:** As a knowledge worker, I want every entity and concept page to include verbatim quotes from the sources that mention them, so that claims are always traceable to their exact origin without relying on paraphrase.

#### Acceptance Criteria

1. THE wiki-schema.md SHALL define the **Mentions in Source** format as: `"Verbatim quote in original language (optional translation)" — [[source-name|display-name]]`.
2. THE Kiro_Agent SHALL populate the **Mentions in Source** section with quotes copied verbatim from the source — paraphrase and summarization are prohibited in this section.
3. THE wiki-schema.md SHALL specify that each quote MUST include a source wiki-link so that future page merges can trace the quote to its origin.
4. WHEN multiple quotes from the same source are present on a page, they SHALL appear together in the same block, separated by newlines, rather than each in a separate section.
5. THE wiki-schema.md SHALL specify that the language of verbatim quotes is always the original source language; an optional translation in parentheses MAY follow the quote.

---

### Requirement 14: Multi-Source Merge Rules

**User Story:** As a knowledge worker, I want the agent to follow explicit merge rules when a second (or later) source touches an existing wiki page, so that no information is lost and the page history remains coherent.

#### Acceptance Criteria

1. THE ingest-workflow.md SHALL specify that WHEN a source being ingested adds information to an existing wiki page, the `sources` frontmatter array SHALL be appended with the new source wiki-link; the existing entries SHALL NOT be overwritten.
2. THE ingest-workflow.md SHALL specify that WHEN a source being ingested introduces an alternative name for an entity or concept already in the wiki, the `aliases` frontmatter field SHALL be appended with the new alias; existing aliases SHALL NOT be removed.
3. THE ingest-workflow.md SHALL specify that WHEN a page has `reviewed: true` set, THE Kiro_Agent SHALL only append genuinely new information from the new source and SHALL NOT alter, rewrite, or replace any existing content on that page.
4. THE ingest-workflow.md SHALL specify that WHEN a source adds nothing new to an existing page (no new facts, aliases, or quotes), THE Kiro_Agent SHALL skip updating that page and MAY emit a `NO_NEW_CONTENT` note in its working summary rather than making a no-op write.
5. WHEN a contradiction is detected during a multi-source ingest, THE Kiro_Agent SHALL record the contradiction in a dedicated `## Contradictions` section on the affected page in addition to any inline callout, preserving both the existing claim and the new contradicting claim with source attribution for each.

---

### Requirement 15: Wiki-Link Syntax

**User Story:** As a knowledge worker, I want all internal cross-references to use wiki-link syntax, so that the wiki is compatible with tools that parse `[[...]]` links for graph traversal and link resolution.

#### Acceptance Criteria

1. THE wiki-schema.md SHALL define the internal linking convention as wiki-link syntax of the form `[[entities/page-name|Display Name]]` or `[[concepts/page-name|Display Name]]` for all cross-references between wiki pages.
2. THE wiki-schema.md SHALL specify that wiki-links use the full path from the `wiki/` root without a leading slash (e.g. `[[entities/plutarch|Plutarch]]`, not `[[../entities/plutarch]]`).
3. THE wiki-schema.md SHALL distinguish between wiki-links (for internal cross-references between wiki pages) and markdown links (for citations pointing to files under `raw/`); the two syntaxes SHALL NOT be used interchangeably.
4. THE Kiro_Agent SHALL use wiki-link syntax in **Related Entities**, **Related Concepts**, and **Mentions in Source** sections, and in the `sources` and `related` frontmatter arrays; plain markdown relative links are reserved for citations only.

---

### Requirement 16: Naming Conventions

**User Story:** As a knowledge worker, I want consistent naming conventions enforced across all wiki pages, so that filenames are predictable and entity names are never inadvertently translated or altered.

#### Acceptance Criteria

1. THE wiki-schema.md SHALL specify that all wiki page filenames use lowercase-with-hyphens (slug) format (e.g. `thomas-hobbes.md`, `spatiotemporal-continuity.md`).
2. THE wiki-schema.md SHALL specify that entity and concept names used within page content and in wiki-links SHALL preserve the original language of the source document and SHALL never be translated by the Kiro_Agent.
3. THE wiki-schema.md SHALL specify that wiki-link display names SHALL match the canonical page title as it appears in the page's `# Heading`.

---

### Requirement 17: Stale Claim Threshold

**User Story:** As a knowledge worker, I want a concrete staleness threshold defined, so that the lint workflow applies a consistent, objective standard when identifying stale claims.

#### Acceptance Criteria

1. THE lint-workflow.md SHALL define the stale claim threshold as **90 days**: a wiki claim is considered potentially stale if a source in `raw/` with a `date` (or `created`) value more than 90 days newer than the source summary page that originally introduced the claim contradicts or supersedes it.
2. THE lint-workflow.md SHALL instruct the Kiro_Agent to use the `date` frontmatter field on source summary pages to compare recency when evaluating staleness.
3. THE wiki-schema.md SHALL document the 90-day stale threshold in the maintenance policies section so it is visible in every session.

---

### Requirement 18: Orphan Page Definition (Inbound Links)

**User Story:** As a knowledge worker, I want orphan pages detected by inbound-link absence, not just index absence, so that pages buried off the index but still linked elsewhere are correctly identified.

#### Acceptance Criteria

1. THE lint-workflow.md SHALL define an orphan page as a wiki page with **no inbound links from any other wiki page** (not merely absent from `wiki/index.md`).
2. THE lint-workflow.md SHALL instruct the Kiro_Agent to scan all pages under `wiki/` for wiki-links and markdown links that reference each page, and flag any page that is referenced by zero other wiki pages as an orphan.
3. THE lint-workflow.md SHALL treat absence from `wiki/index.md` as a separate, co-occurring issue (index drift) rather than the definition of an orphan.
