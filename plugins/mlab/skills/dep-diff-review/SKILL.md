---
name: dep-diff-review
description: Review what a change or pull request introduces into the dependency tree - new packages, version bumps, typosquats, install scripts, vulnerabilities - using postmortem diff plus mlab.sh enrichment. Use when reviewing a PR that touches a lockfile, comparing two branches' dependencies, or gating dependency changes in CI.
---

# Dependency Diff Review

> CLI details and preflight: `${CLAUDE_PLUGIN_ROOT}/docs/postmortem-reference.md`. mlab output shapes: `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`.

The question is never "is this project healthy" but "what does THIS CHANGE introduce". Cost scales with the diff, so `--online --vulns` are cheap here - use them.

## Workflow

### 1. Preflight and target

`postmortem --version`. Then one of:
- GitHub PR: `postmortem diff https://github.com/<owner>/<repo>/pull/<n> --online --vulns --json -o -` (both sides come from the PR)
- Two local states: `postmortem diff <old-dir> <new-dir> --online --vulns --json -o -` (e.g. two worktrees/checkouts)
- Production focus: add `--omit dev` when the review only gates what ships.

### 2. Read the diff

Three buckets: added, removed, version-changed. For each ADDED or version-changed package, the online pass already carries reputation and provenance. Escalate on:
- brand-new or recently transferred packages (`postmortem timeline <pkg>` for npm: handovers, install-script appearance)
- packages that gain an install script in this change (`postmortem scripts <new-dir>` and compare)
- names one edit away from popular packages (typosquat suspicion)
- vulnerabilities the change INTRODUCES (ignore pre-existing ones here; they belong to sbom-audit)

### 3. Enrich what deserves it

Suspicious added package → `postmortem why <pkg> <new-dir> --blast` for what a compromise would reach. Registry/homepage domains that look off → mlab domain/URL scans. Introduced CVEs → KEV/EPSS triage from the results, `actors_by_cve` on anything KEV.

### 4. Verdict for the review

```
## Dep review: <PR/change> - <approve | approve with comments | block>

| Package | Change | Risk | Why |
|---|---|---|---|

### Blocking
<findings that must be resolved, each with the evidence>

### Non-blocking
<worth a comment>

Introduced vulns: <n> (<worst>) | New install scripts: <n> | New maintainers reached: <n>
```

For CI use, point the user at `postmortem scan --sarif` (GitHub Code Scanning) and `postmortem ci` for pipeline templates; a review skill run is the interactive complement, not the gate itself.

## Rules

- Judge the delta, not the repo: pre-existing debt is out of scope and saying so keeps reviews fast.
- "block" needs evidence tied to the change; suspicion without evidence is a comment, not a block.
- Never install or execute the changed dependencies to inspect them.
