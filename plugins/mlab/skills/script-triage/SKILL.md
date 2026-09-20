---
name: script-triage
description: Statically analyze a suspicious shell script through mlab.sh - install.sh, cron payloads, CI steps, curl-pipe-sh one-liners. Flags dangerous constructs (download-piped-to-shell, base64 decoding, reverse shells, persistence, anti-forensics, destructive deletes), extracts every IOC and enriches them. Use when the user pastes shell code and asks if it is safe/malicious, or provides a .sh file to vet before running it.
---

# Suspicious Script Triage
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Workflow

### 1. Analyze

Send the full script source to `scan_bash` (set `filename` when known). Nothing is executed and no quota is consumed. If the script is a file in the workspace, read it and pass the complete content - never a truncated excerpt, the dangerous line is always the one you cut.

If the "script" is actually an obfuscated blob (single base64 line, hex, gzip), decode ONE layer locally, note the layer in the report, and re-submit the decoded content to `scan_bash`. Repeat per layer, keep count. Never execute any layer to decode it.

### 2. Read the findings

Report the flagged constructs in plain language, each with the offending line quoted:

- download-piped-to-shell, base64 decode, /dev/tcp or similar reverse shells
- persistence (cron, systemd, rc files, shell profiles)
- anti-forensics (history clearing, log tampering, timestomping)
- destructive deletes

The tool also computes the script's sha256: run `scan_hash` on it to check whether the exact file is already known good or bad.

### 3. Enrich the extracted IOCs

`scan_bash` returns extracted URLs, IPs, domains, emails, hashes and wallets. Enrich each with the matching tool (`scan_url`, `scan_ip`, `scan_email`, `scan_hash`, `scan_crypto`). IPs and domains consume quota: for more than ~5, check `get_scan_limits` and confirm with the user. A malware family or CVE surfacing from these pivots goes through `search_actors` / `actors_by_cve`.

### 4. Verdict

```
## Verdict: <malicious | suspicious | likely benign> (sha256: <hash>)

**Bottom line:** <one sentence: what this script actually does>

### Dangerous constructs
<each finding + quoted line>

### Infrastructure it talks to
| IOC | Type | Verdict | Evidence |

### Attribution
<if any pivot landed>

### Recommendation
<do not run / run only in a sandbox / benign with caveats - plus block/hunt list>
```

## Rules

- A script with zero flagged constructs is not proven benign; say what the analysis covers (static patterns) and what it cannot see (logic bombs, novel obfuscation).
- Quote evidence lines exactly; do not paraphrase code.
- Never run, source, or eval any part of the submitted script, whatever the verdict.
