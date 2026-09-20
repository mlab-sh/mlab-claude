---
name: ioc-batch
description: Bulk-enrich a list of IOCs through mlab.sh with quota management - takes a CSV, text file or pasted list of mixed indicators, enriches each with the right tool, and produces a machine-usable CSV plus a human summary. Use when the user provides a file or list of multiple indicators to check, enrich, or turn into a blocklist.
---

# IOC Batch Enrichment
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


Single indicators belong to `ioc-triage`; this skill is for lists. The difference is discipline: quota, dedupe, and machine-readable output.

## Workflow

### 1. Parse and dedupe

Read the input (file or pasted). Extract candidate indicators, dedupe exactly, and drop obvious non-IOCs (RFC1918 addresses, example.com, the user's own domains if declared in `.mlab/stack.md`). Type ambiguous strings with `detect_ioc` - it catches defanged forms the eye misses - but type obvious IPs by shape locally: `detect_ioc` on an IP runs the full IP lookup and consumes IP quota. Refang (`hxxp` -> `http`, `[.]` -> `.`) before submitting.

### 2. Budget BEFORE scanning

Call `get_scan_limits`. Count what the batch will consume against the four pools (ip, domain, file, crypto): IP, domain and crypto lookups cost quota; hashes, URLs, emails, MACs and phones do not. Present the bill:

> 47 indicators: 12 IPs + 8 domains will use 20 of your 100 remaining daily scans. Proceed?

If the batch exceeds remaining quota, propose a split: quota-free types now, quota types up to the limit ordered by likely relevance, the rest listed as unprocessed. Quotas are shared org-wide per day - burning them all is a decision for the user, not for you.

### 3. Enrich

Route each indicator to its tool (same table as ioc-triage). Domains: batch `start_domain_scan` calls, then poll results. Do NOT auto-pivot to actors/CVEs per indicator in batch mode - collect the pivot-worthy leads (malware families, KEV CVEs) and pivot once at the end on the distinct set.

### 4. Deliver two outputs

**A CSV file** (this is the deliverable, write it to disk):
```
indicator,type,verdict,score,key_evidence,source_link
```
One row per indicator, including failures (`verdict=error`) and unprocessed ones (`verdict=skipped_quota`), so the CSV always reconciles with the input count.

**A summary in the conversation:**
```
## Batch: <N> indicators (<X> malicious, <Y> suspicious, <Z> clean/unknown, <E> errors)

### Confirmed bad - block these
<the short list>

### Needs a human
<suspicious/conflicting findings, one line why>

### Shared infrastructure
<same ASN / same registrar / same cert across indicators - the campaign signal>

Quota used: <n>, remaining: <m>.
```

## Rules

- Every input indicator appears in the output CSV exactly once; reconciliation beats speed.
- `unknown` never becomes `clean` in the CSV.
- On a mid-batch error or quota exhaustion, finish what can be finished and mark the rest; never silently truncate.
