---
inclusion: manual
---

# Query Workflow

## Activation

This workflow is activated when a user includes `#query-workflow` in their prompt. Once activated, follow the query sequence below to answer the user's question using the wiki.

## Query Sequence

Follow these steps in order for every query operation:

1. **Read `wiki/index.md`.** If it cannot be read for any reason, go to the [Index Unavailable](#index-unavailable) section and halt immediately.
2. **Identify relevant pages.** From the index, select up to 10 pages most relevant to the user's question.
3. **Read those pages.** Read each of the identified pages in full.
4. **Compose an answer.** Write a clear, accurate answer to the user's question. The answer must include at least one citation referencing a specific wiki page used, formatted as `[Page Title](../relative/path.md)`.
5. **Present the answer to the user.**
6. **Prompt the user to confirm filing.** Ask explicitly: "Would you like me to file this answer as a new wiki page?"
7. **Only if the user confirms:** file the answer as a new concept page by following the [Filing an Answer](#filing-an-answer) section.

## Index Unavailable

If `wiki/index.md` cannot be read at the start of a query:

- Notify the user that the wiki index is unavailable and the query cannot proceed.
- Halt immediately.
- Do **not** modify any wiki files.

## Filing an Answer

When the user confirms they want the answer filed:

- **Page location:** `wiki/concepts/<slug>.md` — derive the slug from the page title (lowercase, hyphens for spaces).
- **Required frontmatter:**
  ```yaml
  ---
  type: concept
  related: [<relative links to related wiki pages>]
  ---
  ```
- **Update `wiki/index.md`:** Add an entry for the new page under the `## Concepts` section.
- **Append a Log_Entry to `wiki/log.md`** (see [Log Entry](#log-entry) below).

## Log Entry

A log entry is appended to `wiki/log.md` **only when an answer is filed** as a new wiki page. Append the following as the last line of the file:

```
## [YYYY-MM-DD] query | <page title>
```

Replace `YYYY-MM-DD` with today's date and `<page title>` with the title of the filed concept page.
