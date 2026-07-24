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
