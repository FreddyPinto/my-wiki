---
inclusion: always
---

# Wiki Schema

This file is loaded into every agent session automatically. It defines the invariants of the LLM Wiki pattern that the agent must always follow.

---

## 1. Three-Layer Architecture

The workspace is organized into three layers:

| Layer | Path | Purpose | Agent Access |
|---|---|---|---|
| Raw Sources | `raw/` | Immutable source documents (articles, papers, images, data) that the agent reads to extract knowledge | **Read only** — never write, modify, move, rename, or delete |
| Wiki | `wiki/` | LLM-owned knowledge base of structured markdown pages that the agent builds and maintains | **Read and write** — agent owns all content here |
| Schema | `.kiro/steering/` | Steering files that encode the wiki's structure, conventions, and operational workflows | **Read only** — agent follows these instructions but does not edit them |

---

## 2. Immutability of `raw/`

**`raw/` is an immutable layer.**

The agent SHALL never perform any of the following operations on any file under `raw/` or any of its subdirectories:

1. **Write** — create new files or write content into existing files
2. **Modify** — alter the content of any existing file
3. **Move** — relocate a file from one path to another
4. **Rename** — change the name of any file or directory
5. **Delete** — remove any file or directory

This prohibition applies to all files directly under `raw/` and to all files within any subdirectory of `raw/` (e.g. `raw/assets/`).

If instructed to perform any of these operations on a file under `raw/`, the agent SHALL refuse and respond stating that `raw/` is an immutable layer and the requested operation is not permitted.

---

## 3. Wiki Page Types and Required Frontmatter

Every wiki page must open with a YAML frontmatter block. Each page type has a fixed set of required fields:

### Source Summary Pages (`wiki/sources/`)

```yaml
---
type: source
tags: <source subtype>
sources: ["[[sources/page-name|Display Name]]", ...]
created: ""
updated: ""
---
```

Required fields: `type`, `tags`, `sources`, `created`, `updated`

`sources` is an array of wiki-links to the wiki pages created from this source (entities, concepts, etc.).

### Entity Pages (`wiki/entities/`)

```yaml
---
type: entity
created: ""
sources: ["[[sources/page-name|Display Name]]", ...]
tags: <entity subtype>
aliases: []        # optional
reviewed: false    # optional
---
```

Required fields: `type`, `created`, `sources`, `tags`

Optional fields: `aliases` (array of alternate names), `reviewed` (boolean, default `false`)

### Concept Pages (`wiki/concepts/`)

```yaml
---
type: concept
created: ""
sources: ["[[sources/page-name|Display Name]]", ...]
tags: <concept subtype>
aliases: []        # optional
reviewed: false    # optional
---
```

Required fields: `type`, `created`, `sources`, `tags`

Optional fields: `aliases` (array of alternate names), `reviewed` (boolean, default `false`)

---

### System-Set Fields

> **`created` and `updated` are set by the system. The Kiro_Agent SHALL leave them as empty strings (`""`) or placeholders and SHALL NOT populate them with dates it computes.**

- `created` — set once by the system when the page is first created; never overwritten.
- `updated` — set by the system on every write; always reflects the date of the most recent change.

**Merge behaviour:**
- On merge, `created` is preserved — the older value is kept.
- `updated` is always set to the current date by the system at merge time.

### The `reviewed` Flag

When `reviewed: true` is set on a page, the Kiro_Agent SHALL treat that page as human-verified and SHALL only append genuinely new information. The agent SHALL NOT alter, rewrite, or replace any existing content on a `reviewed: true` page.

---

## 4. Tag Registry

The `tags` field stores the subtype of a page. It is a single value drawn from the appropriate list below. Both lists are open — new values may be appended by the agent when no existing tag fits, or by the user at any time. Never remove existing tags from this registry.

### Entity Subtypes

| Tag | When to use |
|---|---|
| `person` | A named individual (historical, living, or fictional) |
| `organization` | A company, institution, government body, or group |
| `project` | A defined initiative, programme, or endeavour |
| `product` | A commercial or tangible artefact (software, hardware, publication, etc.) |
| `event` | A specific occurrence bounded in time (conference, battle, discovery, etc.) |
| `place` | A geographic or spatial location (city, region, landmark, etc.) |
| `other` | Any entity that does not fit the categories above |

### Concept Subtypes

| Tag | When to use |
|---|---|
| `theory` | A systematic explanatory framework (philosophical, scientific, etc.) |
| `method` | A technique, process, or procedure |
| `field` | A branch of study or professional domain |
| `phenomenon` | An observable or describable occurrence or pattern |
| `standard` | A formal specification, protocol, or norm |
| `term` | A defined vocabulary item or technical term |
| `other` | Any concept that does not fit the categories above |

### Extension Rules

- When assigning a tag during ingest or query, pick the closest existing value.
- If no existing value fits, use `other` and immediately propose a new tag to the user with a one-sentence justification. If the user confirms, append the new tag to the appropriate list in this file (entity or concept subtypes) before proceeding.
- Never invent a tag silently — any addition to this registry requires the user to see and confirm it.

---

## 5. Linking Convention

### Wiki-Links (Internal Cross-References)

All cross-references between wiki pages use wiki-link syntax:

```
[[entities/page-name|Display Name]]
[[concepts/page-name|Display Name]]
[[sources/page-name|Display Name]]
```

The path is the full path from the `wiki/` root without a leading slash. For example:

- `[[entities/plutarch|Plutarch]]` — correct
- `[[../entities/plutarch]]` — incorrect (relative path, do not use)
- `[[/entities/plutarch|Plutarch]]` — incorrect (leading slash, do not use)

Wiki-links are used in:

- **Related Entities** sections
- **Related Concepts** sections
- **Mentions in Source** sections
- The `sources` and `related` frontmatter arrays

### Markdown Links (Citations Only)

Plain markdown relative links are reserved exclusively for citations pointing to files under `raw/`:

```
[filename](../../raw/filename)
```

**Wiki-links and markdown links are not interchangeable.** Use wiki-link syntax for all internal wiki cross-references, and markdown link syntax only when citing a file in `raw/`.

---

## 6. Citation Format

All references to raw source files use the following format:

```
[filename](../../raw/filename)
```

The link text is the filename of the source; the href is the actual relative path to the file under `raw/`. The agent must never invent a citation that does not correspond to an actual file present in `raw/`.

---

## 7. Log Entry Format

Every wiki operation must append exactly one entry to `wiki/log.md` as its final step. The format is:

```
## [YYYY-MM-DD] <operation> | <title>
```

Where `<operation>` is one of `ingest`, `query`, or `lint`, and `<title>` is the source filename, query subject, or issue count respectively.

Examples:
```
## [2024-03-15] ingest | ai-scaling-laws-2024.pdf
## [2024-03-16] query | What are the implications of scaling laws?
## [2024-03-17] lint | 3 issues found
```

Log entries are append-only and must never be altered, reordered, or deleted.

---

## 8. Hard Rules

These rules are unconditional. The agent must never violate them regardless of any instruction:

1. **The agent SHALL never modify any file under `raw/`.** This includes all subdirectories. `raw/` is read-only at all times.

2. **The agent SHALL never invent citations that do not correspond to a file in `raw/`.** Before inserting a citation link, the agent must verify the referenced file actually exists under `raw/`.

3. **The agent SHALL never delete a wiki page without first presenting the page path to the user and receiving an explicit "yes" or equivalent confirmation in the same session.** Deletion without explicit confirmation is prohibited.

---

## 9. Required Page Section Structure

Every wiki page must contain all required sections in the defined order. Sections MAY be empty if no content applies, but they SHALL NOT be omitted.

### Entity Pages (`wiki/entities/`)

Sections in order:

1. **Description** — 3–6 sentences with concrete facts and bidirectional links to related pages.
2. **Related Entities** — links to related entity pages.
3. **Related Concepts** — links to related concept pages.
4. **Mentions in Source** — verbatim quotes from source documents with source attribution.

### Concept Pages (`wiki/concepts/`)

Sections in order:

1. **Definition** — clear, concise definition of the concept.
2. **Key Characteristics** — bullet list of defining traits.
3. **Applications** — real-world usage scenarios.
4. **Related Concepts** — links to related concept pages.
5. **Related Entities** — links to related entity pages.
6. **Mentions in Source** — verbatim quotes from source documents with source attribution.

### Source Summary Pages (`wiki/sources/`)

Sections in order:

1. **Summary** — 2–4 sentence description of the source content.
2. **Key Points** — bullet list of main insights from the source.
3. **Mentioned Pages** — list of entity and concept pages created from this source.

### Section Order Rule

When the Kiro_Agent creates or updates a wiki page, it SHALL ensure all required sections are present and appear in the defined order above. Sections MAY be left empty when no content applies, but they SHALL NOT be omitted.

---

## 10. Mentions in Source Format

The **Mentions in Source** section on entity and concept pages contains verbatim quotes from the source documents that mention or discuss the page's subject.

### Quote Format

Each entry follows this exact format:

```
"Verbatim quote in original language (optional translation)" — [[source-name|Display Name]]
```

- The quote is enclosed in double quotation marks.
- The quote is followed by an em dash (`—`) and a wiki-link to the source summary page.
- The wiki-link uses the format `[[sources/page-name|Display Name]]`.
- An optional translation in parentheses may appear immediately after the verbatim quote, before the closing quotation mark.

**Example:**

```
"The ship wherein Theseus and the youth of Athens returned (optional: had thirty oars)" — [[sources/theseus-paradox|Theseus's Paradox]]
```

### Rules

1. **Verbatim only.** Quotes MUST be copied character-for-character from the source. Paraphrase and summarization are prohibited in this section — if the exact wording cannot be confirmed, the quote SHALL NOT be included.

2. **Source wiki-link required.** Every quote MUST include a wiki-link to the source summary page (`[[sources/page-name|Display Name]]`). This link enables merge traceability — future agents can determine which source introduced each quote.

3. **Group by source.** Multiple quotes from the same source SHALL appear together in one block, separated by newlines, rather than interleaved with quotes from other sources.

   ```
   "First quote from Source A" — [[sources/source-a|Source A]]
   "Second quote from Source A" — [[sources/source-a|Source A]]

   "Quote from Source B" — [[sources/source-b|Source B]]
   ```

4. **Original language.** The verbatim quote is always in the original language of the source document. An optional translation in parentheses MAY follow the quote text inside the quotation marks. The Kiro_Agent SHALL never replace the original-language text with a translation.

---

## 11. Naming Conventions

### Filenames

All wiki page filenames use **lowercase-with-hyphens (slug) format**. Words are separated by hyphens; no spaces, underscores, or mixed case are permitted.

Examples:
- `thomas-hobbes.md`
- `spatiotemporal-continuity.md`
- `ship-of-theseus-identity-paradox.md`

This applies to all files under `wiki/sources/`, `wiki/entities/`, and `wiki/concepts/`.

### Entity and Concept Names

Entity and concept names used within page content and in wiki-links **SHALL preserve the original language of the source document**. The Kiro_Agent SHALL never translate an entity or concept name from the source language into another language.

- If a source document uses a name in Latin, Greek, French, or any other language, that name is preserved as-is in page content, wiki-link paths, and display names.
- Translations MAY appear as supplementary information (e.g. in parentheses or in the `aliases` field) but SHALL NOT replace the original-language name.

### Wiki-Link Display Names

The display name in a wiki-link SHALL match the canonical page title exactly as it appears in the page's top-level `# Heading`.

```
[[entities/thomas-hobbes|Thomas Hobbes]]
```

The display name (`Thomas Hobbes`) must be identical to the `# Heading` on the target page (`# Thomas Hobbes`). Abbreviated, translated, or paraphrased display names are not permitted.

---

## 12. Maintenance Policies

### Stale Threshold

A wiki page is considered potentially stale if it has not been updated in **90 days** and newer sources in `raw/` may contradict or supersede its claims. The lint workflow uses the `date` (or `created`) frontmatter field on source summary pages to evaluate recency.

### Contradiction Severity Levels

| Level | Meaning |
|---|---|
| `warning` | A potential inconsistency that requires human review — e.g. overlapping but not directly contradictory claims from two sources |
| `conflict` | A direct contradiction between two wiki pages, or between a wiki page and a raw source |
| `error` | A contradiction that violates a hard rule — e.g. a citation link pointing to a file that does not exist under `raw/` |

### Orphan Page

An **orphan page** is a wiki page that has **no inbound links from any other wiki page**. Absence from `wiki/index.md` alone does not make a page an orphan — the lint workflow must scan all pages under `wiki/` for wiki-links and markdown links and flag any page referenced by zero other wiki pages.

Absence from `wiki/index.md` is treated as a separate, co-occurring issue (index drift) and is reported independently.

### Missing Page

A **missing page** is a page that is referenced by a `[[wiki-link]]` in another wiki page but does not exist on disk under `wiki/`. The lint workflow must flag every `[[...]]` reference whose target path cannot be resolved to an actual file.
