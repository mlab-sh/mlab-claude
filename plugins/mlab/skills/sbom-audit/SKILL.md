---
name: sbom-audit
description: Audit a dependency lockfile or SBOM for known vulnerabilities via mlab.sh, prioritize findings by KEV status and EPSS, and attribute actively exploited CVEs to threat actors. Use when the user provides a lockfile (package-lock.json, yarn.lock, pnpm-lock.yaml, Cargo.lock, requirements.txt, poetry.lock, uv.lock, Pipfile.lock, go.sum, composer.lock, Gemfile.lock, pom.xml, gradle.lockfile, packages.lock.json, pubspec.lock, mix.lock) or a CycloneDX / SPDX SBOM, or asks to check dependencies for vulnerabilities.
---

# SBOM / Lockfile Vulnerability Audit
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Workflow

### 1. Scan

Send the raw lockfile content to `scan_sbom`. Format is auto-detected; force `format` only if detection fails. Parsing happens server-side; never install or resolve packages locally to "verify".

If the lockfile lives in a repo the user pointed at, read the file and pass its full content, not a summary.

Read `complete` and `coverage_notes` first. A cut manifest, `scan_failures` or `unscanned` entries mean part of the dependency tree was not checked: say so at the top of the report, never present the result as clean.

### 2. Prioritize

Each finding is already one package x one CVE, with `severity`, `cvss` and `fixed_in`. Establish priority in this order (highest signal first):

1. **KEV** (CISA Known Exploited Vulnerabilities) - actively exploited, always P0.
2. **EPSS** - probability of exploitation; > 0.1 is notable, > 0.5 is urgent.
3. **CVSS** - baseline severity, tie-breaker only.

Call `cve_detail` on every critical/high finding that has a `cve` and on anything KEV-listed to get the full vector, EPSS percentile, `kev_due_date` (the CISA deadline - useful pressure in the report) and affected version ranges. Do not fan out `cve_detail` over dozens of LOW findings; summarize those in bulk.

### 3. Attribute what matters

For KEV-listed and high-EPSS CVEs, call `actors_by_cve`. Known exploitation by a named actor changes the remediation conversation ("patch eventually" vs "assume targeted, hunt now").

### 4. Report

```
## SBOM audit: <file> - <N> packages, <M> vulnerabilities

<coverage line when `complete` is false: what was not scanned, from `coverage_notes`>

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
