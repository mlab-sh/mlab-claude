---
name: sbom-audit
description: Audit a dependency lockfile or SBOM for known vulnerabilities via mlab.sh, prioritize findings by KEV status and EPSS, and attribute actively exploited CVEs to threat actors. Use when the user provides a Cargo.lock, package-lock.json, requirements.txt, go.sum, composer.lock, Gemfile.lock or CycloneDX SBOM, or asks to check dependencies for vulnerabilities.
---

# SBOM / Lockfile Vulnerability Audit
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Workflow

### 1. Scan

Send the raw lockfile content to `scan_sbom`. Format is auto-detected; force `format` only if detection fails. Parsing happens server-side; never install or resolve packages locally to "verify".

If the lockfile lives in a repo the user pointed at, read the file and pass its full content, not a summary.

### 2. Prioritize

For each advisory returned, establish severity in this order (highest signal first):

1. **KEV** (CISA Known Exploited Vulnerabilities) - actively exploited, always P0.
2. **EPSS** - probability of exploitation; > 0.1 is notable, > 0.5 is urgent.
3. **CVSS** - baseline severity, tie-breaker only.

Call `cve_detail` on every CRITICAL/HIGH finding and on anything KEV-listed to get the full vector, EPSS percentile, `kev_due_date` (the CISA deadline - useful pressure in the report) and affected version ranges. Do not fan out `cve_detail` over dozens of LOW findings; summarize those in bulk.

### 3. Attribute what matters

For KEV-listed and high-EPSS CVEs, call `actors_by_cve`. Known exploitation by a named actor changes the remediation conversation ("patch eventually" vs "assume targeted, hunt now").

### 4. Report

```
## SBOM audit: <file> - <N> packages, <M> advisories

### P0 - Actively exploited (KEV)
| Package | Version | CVE | EPSS | Fixed in | Known actors |

### P1 - High severity / high EPSS
<same table>

### P2 - The rest, summarized by package

### Remediation plan
<ordered upgrade list: package -> version, grouping upgrades that ship together>

### Hunting notes
<only if actors were found: what exploitation of these CVEs looks like>
```

## Rules

- A vulnerable dependency is only exploitable in context; flag when an advisory clearly targets a feature the project may not use, but never downgrade a KEV finding on that basis.
- Report the advisory's "fixed in" version, not just "upgrade".
- If the lockfile format is unsupported or parsing fails, report the error verbatim; do not hand-roll parsing.
