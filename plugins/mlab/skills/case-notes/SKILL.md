---
name: case-notes
description: Keep a persistent investigation journal - mlab.sh bookmarks hold the case's indicator set (IPs, domains, hashes), a local case file holds the narrative. Use when the user opens/resumes an investigation, says "bookmark this", "add to the case", asks what is in the current case, or wants a case summary.
---

# Investigation Case Notes
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


Two stores, one case:
- **mlab bookmarks** (`add_bookmark` / `get_bookmarks` / `remove_bookmark`): the indicator set. Shared with the whole org and visible in the mlab UI, but flat - only IPs, domains and file hashes, no notes attached.
- **Local case file** `.mlab/cases/<case-slug>.md`: the narrative - timeline, hypotheses, verdicts, and the indicators that bookmarks cannot hold (URLs, emails, wallets, phone numbers).

## Workflow

### Opening / resuming

On "open a case" create the case file with a header (name, date, one-line scope). On "resume", read the case file, then `get_bookmarks` and `get_scan_history` to see what happened platform-side since - other analysts' scans show up there. Summarize the delta in two lines.

### During the investigation

When a finding is worth keeping ("bookmark this", or a confirmed-bad indicator during any other skill's run):
- IP, domain or hash → `add_bookmark` AND a line in the case file.
- Any other type → case file only, note that mlab cannot bookmark it.

Case file lines are append-only and dated:
```
- 2026-09-20 14:32 [ioc] 185.220.101.1 - TOR exit, C2 for sample X (scan: <link>)
- 2026-09-20 14:40 [hypothesis] initial access via the npm package, not phishing
```

### Closing / summarizing

On "summarize the case" or "close it", produce the summary from the case file plus a fresh `get_bookmarks`, then offer two cleanups: hand the summary to `intel-report` for a shareable document, and `remove_bookmark` on indicators that turned out benign (bookmarks are org-shared; a stale benign bookmark pollutes everyone's view).

## Rules

- Bookmarks are org-wide state, not your scratchpad: never bulk-remove bookmarks you did not add in this case without asking.
- Every bookmark added must exist in the case file too - the file is the source of truth for WHY something is bookmarked.
- One case file per case; parallel investigations never share a file.
