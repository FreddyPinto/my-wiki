---
inclusion: manual
---

# Lint Workflow

## Activation

This steering file is activated manually. To start a lint run, include `#lint-workflow` in your prompt. For example:

> `#lint-workflow` Run a full wiki health check.

The `lint-wiki` hook also activates this workflow when triggered from the Kiro interface.

---

## Lint Sequence

Before running any checks, read `wiki/index.md`. If it is missing or unreadable, report this to the user as a **blocking issue** and stop immediately — do not proceed with any other checks.

If the index is available, run all five checks in order. Complete all checks before presenting results.

### Check 1 — Contradictions

Scan all wiki pages for claims that directly contradict claims in other wiki pages. For each contradiction found, record:
- The two pages involved
- The conflicting claims

### Check 2 — Stale Claims

Compare wiki claims against source dates in `raw/`. A claim is stale when a newer source in `raw/` contradicts it. Use the `date` frontmatter field on source summary pages to determine recency. For each stale claim found, record:
- The wiki page containing the stale claim
- The newer raw source that contradicts it

### Check 3 — Orphan Pages

Identify wiki pages that are not linked from `wiki/index.md`. A page is an orphan if its path does not appear under any section (`## Sources`, `## Entities`, or `## Concepts`) in the index. For each orphan found, record:
- The page path

### Check 4 — Missing Cross-References

Identify pairs of wiki pages that share entities or concepts but do not link to each other. For each missing cross-reference found, record:
- The two pages that should link to each other
- The shared entity or concept

### Check 5 — Data Gaps

Identify topics, entities, or concepts that are mentioned in wiki pages but do not have their own dedicated page under `wiki/entities/` or `wiki/concepts/`. For each data gap found, record:
- The topic name
- The page(s) that mention it

---

## Issue Handling

After completing all five checks, present a **consolidated list** of every issue found, grouped by check. Include the full details recorded for each issue.

**Do not modify any wiki page until the user gives explicit confirmation.** Wait for the user to review the list and confirm which issues (if any) they want you to address before making any changes.

If the index was unreadable (blocking issue), skip directly to the log entry step.

---

## Log Entry

Append a log entry to `wiki/log.md` upon completing a lint operation — whether the run finished all five checks or was terminated early:

```
## [YYYY-MM-DD] lint | <N> issues found
```

Where `<N>` is the total number of issues found across all checks. If the run was blocked by an unreadable index before any checks ran, use `<N> = 0` and note the blocking condition in the same entry if desired.
