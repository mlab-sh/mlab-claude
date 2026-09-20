---
name: threat-watch
description: Generate a threat landscape digest from mlab.sh - recent critical/KEV CVEs filtered to a declared technology stack, with actor attribution. Use when the user asks what is new in vulnerabilities, wants a daily/weekly threat brief, asks about recent CVEs for a product, or sets up a recurring watch on their stack.
---

# Threat Watch
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Stack definition

The watch is only as good as the stack it filters on. Look for a stack file at `.mlab/stack.md` in the working directory (one product or vendor per line, `#` comments allowed). If absent, ask the user for their key products once, then offer to save the file so future runs are zero-question.

Example `.mlab/stack.md`:
```
# edge
nginx
fortinet
# platform
kubernetes
postgresql
gitlab
```

## Workflow

### 1. Sweep

For each stack entry, `cve_search` with the product as query, `date_start` set to the last run (or 7 days back on a first run), one pass with `severity: CRITICAL` and one with `severity: HIGH`. Page through results if the first page is full. Note the run date; suggest the user keep it in `.mlab/stack.md` as a comment for next time.

### 2. Prioritize

Same ladder as everywhere: KEV first, then EPSS, then CVSS - and `cve_search` results already carry `in_kev` and `epss_score`, so this triage happens on the search results directly. `cve_detail` only on the shortlist (for the vector, `kev_due_date` and references), then `actors_by_cve` for who exploits it. One or two `get_actor` calls on the most relevant actor give the digest its "so what".

### 3. Digest

```
# Threat watch - <date> (window: <since>)

**Bottom line:** <one sentence: patch X now / quiet week>

## Act now
| CVE | Product | CVSS | EPSS | KEV | Exploited by | Fix |

## On the radar
<HIGH findings, one line each>

## Actor spotlight
<only when attribution landed: 2-3 sentences on the actor most relevant to this stack>

## No findings for
<stack entries with nothing new - silence is signal too>
```

### 4. Recurring runs

If the user wants this on a schedule, set it up with the session's scheduled-task tooling and make the prompt self-contained: include the stack list and the output shape in the task's own prompt, since a scheduled run starts fresh.

## Rules

- Filter honestly: a CVE in a product the stack does not list does not enter the digest to make it look fuller.
- "No new CVEs" is a valid, useful digest; never pad.
- The digest reports published CVEs, not live exploitation against the user; keep the verbs straight.
