---
inclusion: manual
---

# Ingest Workflow

## Activation

This workflow is activated by typing `#ingest-workflow` in a prompt, or automatically when the `ingest-on-new-source` hook fires on a newly created file under `raw/`.

To activate manually, include `#ingest-workflow` in your message along with the path to the source file you want to ingest. For example:

> `#ingest-workflow` Please ingest `raw/my-document.pdf`

---

## Ingest Sequence

Follow these steps in order. Do not skip or reorder them.

1. **Read the source** — Read the specified file from `raw/`. Do not modify it. If the file cannot be read, notify the user and halt.

2. **Discuss key takeaways** — Present a summary of the source to the user and discuss the most important facts, entities, and concepts it introduces or updates. Confirm understanding before proceeding.

3. **Create a source summary page** — Create a new page under `wiki/sources/` summarising the source. The page must open with the following frontmatter:

   ```yaml
   ---
   type: source
   source: <original filename in raw/>
   date: <YYYY-MM-DD>
   ---
   ```

   The body should summarise the source content, including key facts, entities, and concepts introduced.

4. **Update `wiki/index.md`** — Add a markdown link to the new source summary page under the `## Sources` section. Replace `_No pages yet._` if it is the only entry.

5. **Update entity and concept pages** — Identify all entity and concept pages that mention entities or concepts directly introduced or materially updated by this source. Update each relevant page to reflect the new information. Observe the page update budget (see below).

6. **Flag contradictions** — Before saving any updates, check whether the source contradicts any existing wiki content. If contradictions are found, apply the contradiction handling procedure (see below) rather than silently overwriting.

7. **Append a log entry** — As the final step, append a `Log_Entry` to `wiki/log.md` in the format:

   ```
   ## [YYYY-MM-DD] ingest | <source filename>
   ```

---

## Page Update Budget

- **Populated wiki (5 or more applicable pages exist):** Update between **5 and 15** entity and concept pages per ingest. Prioritise pages most directly affected by the new source.
- **Sparse wiki (fewer than 5 applicable pages exist):** Update **all applicable pages** without error or restriction. Do not skip any applicable page simply because the total count is below 5.

An "applicable" page is one that mentions an entity or concept directly introduced or materially updated by the source being ingested.

---

## Contradiction Handling

When the new source contradicts an existing claim on a wiki page, do **not** silently overwrite the existing content. Instead:

1. Leave the existing claim in place.
2. Insert a callout block immediately after the contradicted claim:

   ```markdown
   > **Contradiction:** Existing claim: "<existing claim text>". New source states: "<new claim text>". See [<source filename>](../../raw/<source filename>).
   ```

3. Do not resolve the contradiction unilaterally. Flag it for human review and move on.

Multiple contradictions on the same page each get their own callout block adjacent to the relevant claim.

---

## Log Entry

The final step of every ingest operation is appending a log entry to `wiki/log.md`. Append after the last existing line; never alter, reorder, or delete any existing entry.

Format:

```
## [YYYY-MM-DD] ingest | <source filename>
```

Example:

```
## [2024-03-15] ingest | ai-scaling-laws-2024.pdf
```

---

## Multi-Source Merge Rules

When a source being ingested touches an existing wiki page, apply the following rules in addition to the standard update procedure.

### 1. `sources` Array — Append Only

When appending information from a new source to an existing page, add the new source wiki-link to the `sources` frontmatter array. **Never overwrite or remove existing entries.**

```yaml
# Before
sources: ["[[sources/theseus-paradox|Theseus's Paradox]]"]

# After ingesting a second source
sources:
  - "[[sources/theseus-paradox|Theseus's Paradox]]"
  - "[[sources/new-source|New Source Title]]"
```

### 2. `aliases` — Append Only

When a new source introduces an alternative name for an entity or concept already in the wiki, append the new alias to the `aliases` frontmatter field. **Never remove existing aliases.**

```yaml
# Before
aliases: ["Ship of Theseus"]

# After finding a new alias in a second source
aliases: ["Ship of Theseus", "Theseus's Ship"]
```

### 3. `reviewed: true` Pages — Append Only

When a page has `reviewed: true` set, the Kiro_Agent SHALL:

- **Only append** genuinely new information from the new source.
- **Never alter, rewrite, or replace** any existing content on that page.
- Add new facts, quotes, and sections after existing content — never in-place.

This rule applies to all sections of the page, including frontmatter arrays, body text, and the Mentions in Source section.

### 4. `NO_NEW_CONTENT` — Skip No-Op Updates

When a source adds nothing new to an existing page (no new facts, no new aliases, no new quotes), the Kiro_Agent SHALL:

1. Skip updating that page entirely — do not make a no-op write.
2. Note `NO_NEW_CONTENT` for that page in the working summary presented to the user.

Example working summary note:
```
- wiki/entities/theseus.md — NO_NEW_CONTENT (source adds no new facts, aliases, or quotes)
```

### 5. Contradictions — Dedicated Section + Inline Callout

When a contradiction is detected during a multi-source ingest, the Kiro_Agent SHALL record it in **two places** on the affected page:

**a) Inline callout** — immediately after the contradicted claim (existing behaviour):

```markdown
> **Contradiction:** Existing claim: "<existing claim text>". New source states: "<new claim text>". See [<source filename>](../../raw/<source filename>).
```

**b) Dedicated `## Contradictions` section** — at the bottom of the page, listing all contradictions found on that page with full source attribution for both sides:

```markdown
## Contradictions

- **<brief label>:** [[sources/first-source|First Source]] states "<existing claim>"; [[sources/second-source|Second Source]] states "<contradicting claim>".
```

If the page already has a `## Contradictions` section, append new entries to it — do not replace the existing ones.
