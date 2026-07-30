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
source: <original filename in raw/>
date: <YYYY-MM-DD>
---
```

Required fields: `type`, `source`, `date`

### Entity Pages (`wiki/entities/`)

```yaml
---
type: entity
tags: <entity subtype>
aliases: [<alternate name>, ...]
---
```

Required fields: `type`, `tags`, `aliases`

### Concept Pages (`wiki/concepts/`)

```yaml
---
type: concept
tags: <concept subtype>
related: [<relative link to related page>, ...]
---
```

Required fields: `type`, `tags`, `related`

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

All cross-references between wiki pages use standard markdown links with relative paths:

```
[Page Title](../relative/path.md)
```

This format must be used consistently for all internal links. Never use absolute paths or bare filenames for cross-references.

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
