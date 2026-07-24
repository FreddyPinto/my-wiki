# Design Document: LLM Wiki Setup

## Overview

The LLM Wiki Setup feature bootstraps the LLM Wiki pattern in a Kiro workspace. The pattern replaces one-shot retrieval-augmented generation (RAG) with a persistent, incrementally maintained knowledge base. Instead of re-deriving knowledge on every query, the Kiro agent compounds knowledge across sessions by maintaining a structured directory of markdown files.

The setup produces no application code. All deliverables are configuration artifacts: a directory scaffold, three steering files encoding the wiki's schema and operational workflows, and two hook files wiring agent automation to IDE events.

---

## Architecture

The system is organized into three layers:

```
workspace/
├── raw/                        ← Layer 1: Immutable source documents
│   └── assets/                 ← Binary and media files
├── wiki/                       ← Layer 2: LLM-owned knowledge base
│   ├── sources/                ← Source summary pages
│   ├── entities/               ← Entity pages
│   ├── concepts/               ← Concept pages
│   ├── index.md                ← Content catalog
│   └── log.md                  ← Append-only operation log
└── .kiro/
    ├── steering/               ← Layer 3: Schema (agent instructions)
    │   ├── wiki-schema.md      ← Always-on constitution
    │   ├── ingest-workflow.md  ← Manual: #ingest-workflow
    │   ├── query-workflow.md   ← Manual: #query-workflow
    │   └── lint-workflow.md    ← Manual: #lint-workflow
    └── hooks/
        ├── ingest-on-new-source.kiro.hook
        └── lint-wiki.kiro.hook
```

### Layer Responsibilities

| Layer | Path | Agent Access |
|---|---|---|
| Raw Sources | `raw/` | Read only — never write, modify, move, rename, or delete |
| Wiki | `wiki/` | Read and write — agent owns all content here |
| Schema | `.kiro/steering/` | Read only — agent follows instructions, does not edit steering files |

### Three Operations

The system drives all agent activity through three named operations:

- **Ingest** — Read a new source from `raw/`, extract knowledge, and integrate it into the wiki.
- **Query** — Answer a user question by reading the wiki, with an option to file the answer as a new concept page.
- **Lint** — Perform a health check: find contradictions, stale claims, orphan pages, missing cross-references, and data gaps.

Each operation is governed by a dedicated steering file and terminates by appending a `Log_Entry` to `wiki/log.md`.

---

## Components and Interfaces

### Directory Scaffold

The initialization step creates the following directories and seed files if they do not already exist. It is idempotent — running it on an already-initialized workspace must not overwrite or reset any existing content.

| Path | Type | Created by init |
|---|---|---|
| `raw/` | Directory | Yes |
| `raw/assets/` | Directory | Yes |
| `wiki/` | Directory | Yes |
| `wiki/sources/` | Directory | Yes |
| `wiki/entities/` | Directory | Yes |
| `wiki/concepts/` | Directory | Yes |
| `wiki/index.md` | File | Yes — see §Wiki Index |
| `wiki/log.md` | File | Yes — see §Wiki Log |

---

### wiki-schema.md — The Constitution

**Path:** `.kiro/steering/wiki-schema.md`  
**Inclusion:** `always`

This file is loaded into every agent session automatically. It encodes the invariants the agent must never violate.

#### Frontmatter

```yaml
---
inclusion: always
---
```

#### Content Sections

1. **Three-Layer Architecture** — Names each layer (`raw/`, `wiki/`, Schema), describes its purpose, and states agent read/write access per layer.

2. **Immutability of `raw/`** — A dedicated section that explicitly labels `raw/` as immutable and enumerates the five prohibited operations: write, modify, move, rename, delete. Applies to all files under `raw/` and all subdirectories.

3. **Wiki Page Types and Required Frontmatter** — Defines the three page types and their mandatory YAML frontmatter fields:

   | Page Type | Required Frontmatter Fields |
   |---|---|
   | Source Summary (`wiki/sources/`) | `type: source`, `source: <filename>`, `date: <YYYY-MM-DD>` |
   | Entity (`wiki/entities/`) | `type: entity`, `aliases: [...]` |
   | Concept (`wiki/concepts/`) | `type: concept`, `related: [...]` |

4. **Linking Convention** — All cross-references between wiki pages use standard markdown links with relative paths:
   ```
   [Page Title](../relative/path.md)
   ```

5. **Citation Format** — All references to raw source files use:
   ```
   [filename](../../raw/filename)
   ```
   The agent must never invent a citation that does not correspond to an actual file in `raw/`.

6. **Log Entry Format** — Every operation appends one entry to `wiki/log.md` in the format:
   ```
   ## [YYYY-MM-DD] <operation> | <title>
   ```

7. **Hard Rules** — Three unconditional prohibitions stated explicitly:
   - The agent SHALL never modify any file under `raw/`.
   - The agent SHALL never invent citations that do not correspond to a file in `raw/`.
   - The agent SHALL never delete a wiki page without first presenting the page path to the user and receiving an explicit "yes" or equivalent confirmation in the same session.

---

### ingest-workflow.md — Ingest Workflow

**Path:** `.kiro/steering/ingest-workflow.md`  
**Inclusion:** `manual`  
**Activation tag:** `#ingest-workflow`

Activated when a user types `#ingest-workflow` in a prompt, or when the `ingest-on-new-source` hook fires.

#### Frontmatter

```yaml
---
inclusion: manual
---
```

#### Content Sections

1. **Activation** — States the tag (`#ingest-workflow`) and how to activate.

2. **Ingest Sequence** — The agent follows these steps in order:
   1. Read the specified source file from `raw/`.
   2. Discuss key takeaways with the user.
   3. Create a source summary page in `wiki/sources/` with required frontmatter (`type`, `source`, `date`).
   4. Update `wiki/index.md` to catalog the new source summary page under `## Sources`.
   5. Identify entity and concept pages that mention entities or concepts directly introduced or materially updated by the source. Update between 5 and 15 pages on a wiki with sufficient existing content; update all applicable pages when fewer than 5 applicable pages exist.
   6. Flag any contradictions found (see contradiction handling below).
   7. Append a `Log_Entry` to `wiki/log.md` as the final step.

3. **Page Update Budget** — On a wiki with sufficient existing content, the agent updates 5–15 pages per ingest. When fewer than 5 applicable pages exist, the agent updates all applicable pages without error.

4. **Contradiction Handling** — When a new source contradicts existing wiki content, the agent marks the affected page with a callout block rather than silently overwriting:
   ```markdown
   > **Contradiction:** Existing claim: "…". New source states: "…". See [source](../../raw/filename).
   ```
   The agent does not resolve contradictions unilaterally — it flags them for human review.

5. **Log Entry** — The final step of every ingest is appending to `wiki/log.md`:
   ```
   ## [YYYY-MM-DD] ingest | <source filename>
   ```

---

### query-workflow.md — Query Workflow

**Path:** `.kiro/steering/query-workflow.md`  
**Inclusion:** `manual`  
**Activation tag:** `#query-workflow`

#### Frontmatter

```yaml
---
inclusion: manual
---
```

#### Content Sections

1. **Activation** — States the tag (`#query-workflow`) and how to activate.

2. **Query Sequence** — The agent follows these steps in order:
   1. Read `wiki/index.md`. If it cannot be read, notify the user and halt — do not modify any wiki files.
   2. Identify up to 10 relevant pages from the index.
   3. Read those pages.
   4. Compose an answer with at least one citation referencing the specific wiki pages used (format: `[Page Title](../relative/path.md)`).
   5. Present the answer to the user.
   6. Explicitly prompt the user: "Would you like me to file this answer as a new wiki page?"
   7. Only if the user confirms: save the answer as a new concept page under `wiki/concepts/`, update `wiki/index.md` under `## Concepts`, and append a `Log_Entry` to `wiki/log.md`.

3. **Index Unavailable** — If `wiki/index.md` cannot be read at the start of a query, the agent notifies the user and halts without modifying any wiki files.

4. **Filing an Answer** — When filing an answer:
   - Page location: `wiki/concepts/<slug>.md`
   - Required frontmatter: `type: concept`, `related: [...]`
   - Update `wiki/index.md` under `## Concepts`
   - Append `Log_Entry`: `## [YYYY-MM-DD] query | <page title>`

5. **Log Entry** (only when an answer is filed):
   ```
   ## [YYYY-MM-DD] query | <page title>
   ```

---

### lint-workflow.md — Lint Workflow

**Path:** `.kiro/steering/lint-workflow.md`  
**Inclusion:** `manual`  
**Activation tag:** `#lint-workflow`

#### Frontmatter

```yaml
---
inclusion: manual
---
```

#### Content Sections

1. **Activation** — States the tag (`#lint-workflow`) and how to activate.

2. **Lint Sequence** — The agent checks for five issue categories in order:
   1. **Contradictions** — Claims in one wiki page that contradict claims in another.
   2. **Stale claims** — Wiki claims contradicted by a newer source in `raw/` (compare source dates).
   3. **Orphan pages** — Wiki pages not linked from `wiki/index.md`.
   4. **Missing cross-references** — Pages that share entities or concepts but do not link to each other.
   5. **Data gaps** — Topics mentioned in wiki pages that lack their own dedicated pages.

3. **Issue Handling** — If `wiki/index.md` is missing or unreadable at the start of a lint run, report this to the user as a blocking issue and stop before any other checks. Otherwise, complete all five checks, then present a consolidated list of all issues found and wait for explicit user confirmation before modifying any wiki pages.

4. **Log Entry** — Appended upon completing a lint operation (whether finished fully or terminated early by the user):
   ```
   ## [YYYY-MM-DD] lint | <N> issues found
   ```

---

### ingest-on-new-source.kiro.hook — Ingest Hook

**Path:** `.kiro/hooks/ingest-on-new-source.kiro.hook`  
**Event:** `fileCreated`  
**File pattern:** `raw/**/*`

#### Behavior

When a new file is created anywhere under `raw/` (including subdirectories), the hook triggers the Kiro agent to:

1. Activate the `#ingest-workflow` steering file.
2. Pass the newly created file's path as the source to ingest.

This makes ingest automatic — the knowledge worker simply drops a file into `raw/` and the agent begins integrating it without any manual prompt.

#### Hook Configuration

```json
{
  "event": "fileCreated",
  "filePattern": "raw/**/*",
  "action": "askAgent",
  "prompt": "A new source file has been added at {{filePath}}. Activate #ingest-workflow and ingest this file into the wiki."
}
```

---

### lint-wiki.kiro.hook — Lint Hook

**Path:** `.kiro/hooks/lint-wiki.kiro.hook`  
**Event:** `userTriggered`

#### Behavior

When the user manually triggers this hook from the Kiro interface, the agent:

1. Activates the `#lint-workflow` steering file.
2. Runs all five lint checks: contradictions, stale claims, orphan pages, missing cross-references, and data gaps.

#### Hook Configuration

```json
{
  "event": "userTriggered",
  "action": "askAgent",
  "prompt": "Activate #lint-workflow and run a full wiki health check covering all five checks: contradictions, stale claims, orphan pages, missing cross-references, and data gaps."
}
```

---

## Data Models

### Wiki Page Frontmatter

All wiki pages open with a YAML frontmatter block. Each page type has a fixed set of required fields.

#### Source Summary Page

```yaml
---
type: source
source: <original filename in raw/>
date: <YYYY-MM-DD>
---
```

Example: `wiki/sources/ai-scaling-laws-2024.md`
```yaml
---
type: source
source: ai-scaling-laws-2024.pdf
date: 2024-03-15
---
```

#### Entity Page

```yaml
---
type: entity
aliases: [<alternate name>, ...]
---
```

Example: `wiki/entities/openai.md`
```yaml
---
type: entity
aliases: [OpenAI, Open AI, oai]
---
```

#### Concept Page

```yaml
---
type: concept
related: [<relative link to related page>, ...]
---
```

Example: `wiki/concepts/scaling-laws.md`
```yaml
---
type: concept
related: ["../entities/openai.md", "../sources/ai-scaling-laws-2024.md"]
---
```

### Log Entry Format

Every log entry follows this exact format:

```
## [YYYY-MM-DD] <operation> | <title>
```

Where `<operation>` is one of `ingest`, `query`, or `lint`.

Examples:
```
## [2024-03-15] ingest | ai-scaling-laws-2024.pdf
## [2024-03-16] query | What are the implications of scaling laws?
## [2024-03-17] lint | 3 issues found
```

---

## wiki/index.md — Initial Content

```markdown
# Wiki Index

This file catalogs all pages in the wiki. The Kiro agent updates it automatically
during ingest and query operations. Pages are organized by type.

## Sources

_No pages yet._

## Entities

_No pages yet._

## Concepts

_No pages yet._
```

---

## wiki/log.md — Initial Content

```markdown
# Wiki Log

This file is an append-only chronological record of all wiki operations performed
by the Kiro agent. Each entry is appended at the end of this file and must never
be altered, reordered, or deleted.

Log entry format:

\`\`\`
## [YYYY-MM-DD] <operation> | <title>
\`\`\`
```

---

## Error Handling

Since this is a configuration-only feature, "error handling" means agent behavior rules encoded in the steering files.

| Situation | Agent Behavior |
|---|---|
| `wiki/index.md` unreadable at query start | Notify user, halt, make no changes |
| `wiki/index.md` unreadable at lint start | Report as blocking issue, stop before other checks |
| Contradiction detected during ingest | Add `> **Contradiction:**` callout, do not overwrite |
| Fewer than 5 applicable pages during ingest | Update all applicable pages (no error) |
| Instruction to write/modify/delete under `raw/` | Refuse, state `raw/` is immutable |
| Request to delete a wiki page | Present path to user, require explicit confirmation |
| Citation invented without corresponding `raw/` file | Prohibited — agent must never do this |

---

## Testing Strategy

This feature produces only configuration files. There is no application code to test with unit or property-based tests. Testing is therefore structural and behavioral:

**Structural verification** — After initialization, verify that all required paths exist with the correct content:
- All directories: `raw/`, `raw/assets/`, `wiki/`, `wiki/sources/`, `wiki/entities/`, `wiki/concepts/`
- All seed files: `wiki/index.md`, `wiki/log.md`
- All steering files with correct frontmatter inclusion values
- Both hook files with correct event types and file patterns

**Idempotency check** — Run initialization twice on the same workspace and confirm no existing file was modified (compare checksums or timestamps).

**Content review** — Manually read each steering file to confirm:
- `wiki-schema.md` covers all three layers, all frontmatter requirements, all hard rules, and the immutability section
- `ingest-workflow.md` covers the full ingest sequence including contradiction handling and log entry
- `query-workflow.md` covers the full query sequence including the index-unavailable halt and the file-answer confirmation prompt
- `lint-workflow.md` covers all five checks, consolidated issue presentation, and log entry

**Hook wiring review** — Confirm each hook specifies the correct event type, file pattern (where applicable), and references the correct workflow activation tag.

Property-based testing is not applicable to this feature. The deliverables are declarative configuration artifacts whose correctness is verified by inspection and structural checks, not by running functions over generated inputs.
