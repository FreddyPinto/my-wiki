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

**Stale threshold: 90 days.**

A wiki claim is considered potentially stale when a source in `raw/` with a `date` (or `created`) value more than 90 days newer than the source summary page that originally introduced the claim contradicts or supersedes it.

To evaluate staleness, use the `date` frontmatter field on source summary pages to compare recency: find the `date` of the source summary page that introduced each claim, then check whether any raw source with a `date` more than 90 days later contradicts or supersedes that claim.

For each stale claim found, record:
- The wiki page containing the stale claim
- The source summary page that originally introduced the claim (and its `date`)
- The newer raw source that contradicts or supersedes it (and its `date`)

### Check 3 — Orphan Pages

An **orphan page** is a wiki page with no inbound links from any other wiki page. Absence from `wiki/index.md` alone does not make a page an orphan — that is a separate **index drift** issue (see below).

To identify orphan pages:
1. Enumerate all wiki pages under `wiki/sources/`, `wiki/entities/`, and `wiki/concepts/`.
2. For each page, scan every other wiki page for wiki-links (`[[...]]`) and markdown links (`[...](...)`) that reference it.
3. A page is an orphan if zero other wiki pages contain any link pointing to it.

For each orphan found, record:
- The page path

**Index drift (co-occurring issue):** Also flag any wiki page that is absent from `wiki/index.md` as an index drift issue — record it separately from the orphan list. A page may be both an orphan and missing from the index, or only one of the two. Report each condition independently.

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
