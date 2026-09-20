---
name: supply-chain-audit
description: Full supply-chain audit of a project - postmortem detects malicious code, install hooks and risky dependencies locally, then mlab.sh enriches what it finds (hashes, endpoints, actors, CVEs). Use when the user asks to audit a project's dependencies or supply chain, check a repo for malicious packages, or vet a codebase before adoption or deployment.
---

# Supply-Chain Audit

> CLI details and preflight: `${CLAUDE_PLUGIN_ROOT}/docs/postmortem-reference.md`. mlab output shapes: `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`.

Local detection + cloud attribution. postmortem finds the problem in the code; mlab tells you who and what is behind it.

## Workflow

### 1. Preflight

`postmortem --version` (see the reference for install fallbacks). Confirm the target: a project directory with lockfiles, or a container image (`--image <ref>`).

### 2. Audit

```
postmortem audit <path> --online --vulns --json -o -
```

`--online` adds source-repo reputation, `--vulns` known advisories (served by vuln.mlab.sh - do not re-run `scan_sbom` afterwards, same data). If the network is unavailable, run offline and say the reputation/vulns columns are missing. If the verdict needs finding-level detail, follow with `postmortem scan <path> --json -o -` for the exact findings (file, pattern, severity).

On a repo containing test fixtures or malware samples, mention `--allow-test-files` exists but keep it off unless the user asks - default output excludes fixture noise deliberately.

### 3. Pivot the findings through mlab

For each `scan`/`audit` finding:
- Artifact hash → `scan_hash` (known family? `mlab_sample`?)
- Embedded URL / IP / domain → `scan_url` / `scan_ip` / domain scan (quota rules apply - budget with `get_scan_limits` on big finding sets)
- Malware family named anywhere → `search_actors` → `get_actor` for TTPs
- KEV/high-EPSS CVEs from `--vulns` → `cve_detail` + `actors_by_cve`
- The package at the center of a finding → `postmortem why <pkg> <path> --blast` for compromise reach, and `postmortem timeline <pkg>` (npm only) for handover/install-script history

### 4. Report

```
## Supply-chain audit: <project> - verdict <CLEAN | ISSUES | CRITICAL>

**Bottom line:** <one sentence>

### Malicious code findings
<per finding: package, file, pattern, severity + the mlab enrichment of its IOCs>

### Vulnerabilities
<KEV/EPSS-prioritized, same ladder as sbom-audit>

### Risky dependencies
<reputation flags, install-script surprises (postmortem scripts), blast radius of the worst offenders>

### Attribution
<actors/families, confidence, sources>

### Actions
<remove/pin/upgrade per package; hunt list if malicious code was found - a malicious dep that ran at install is an incident, not a finding: say so plainly>
```

## Rules

- postmortem exit codes are a CI gate (non-zero = findings at/above `--severity`), not an error - read the output, not just the code.
- A CLEAN offline verdict covers static malware patterns only; without `--online`/`--vulns` say what was not checked.
- Never `npm install` / `pip install` / build the audited project to "verify" - the audit exists because the code is untrusted.
