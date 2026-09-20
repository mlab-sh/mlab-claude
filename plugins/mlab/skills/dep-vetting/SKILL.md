---
name: dep-vetting
description: Vet a single package BEFORE adding it as a dependency - reputation, typosquat check, maintainer history, install scripts, blast radius - using postmortem and mlab.sh. Use when the user asks "can I trust this package", considers adding a dependency (npm i / cargo add / pip install X), or compares candidate libraries.
---

# Dependency Vetting

> CLI details and preflight: `${CLAUDE_PLUGIN_ROOT}/docs/postmortem-reference.md`. mlab output shapes: `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`.

The package is NOT installed yet and must not be during vetting. Everything below inspects it from the outside.

## Workflow

### 1. Preflight

`postmortem --version`. Identify the exact package name and ecosystem the user means - one typo here and you vet the wrong (or the squatting) package.

### 2. Typosquat first

Before anything else: is THIS name the package they think it is? Compare against the popular package it resembles (postmortem ships offline typosquat corpora per ecosystem and flags this in scans; for a not-yet-added package, check manually: edit distance to famous names, download counts wildly lower than the lookalike, recent creation date). A hit here ends the vetting - report the squat.

### 3. History and provenance

- npm: `postmortem timeline <pkg>` - maintainer handovers, when install scripts appeared, repository moves. A handover immediately followed by a new install script is the classic takeover shape.
- Add the package to a THROWAWAY manifest in a temp dir (manifest + lockfile resolution only, never an install with scripts): `mkdir /tmp/vet && cd /tmp/vet && npm i --package-lock-only <pkg>` (or ecosystem equivalent), then `postmortem scan .`, `postmortem tree . --online`, `postmortem scripts .` - what does it pull in, who controls that subtree (`tree --online --human`), does anything run at install?
- `postmortem why <pkg> /tmp/vet --blast` for what a future compromise would reach in the user's real project shape.

### 4. mlab pivots

- Package homepage/repository domains → `scan_url`; young or oddly registered domains are a flag.
- Known advisories → the throwaway lockfile through `scan_sbom`, KEV/EPSS triage.
- Any malware family or IOC surfacing → standard pivots (`scan_hash`, `search_actors`).

### 5. Verdict

```
## Vetting: <pkg>@<version> (<ecosystem>) - <adopt | adopt with pins | avoid>

| Check | Result |
|---|---|
| Typosquat | |
| Maintainer history | |
| Install scripts | |
| Transitive tree (size, controllers) | |
| Known vulns | |
| Static scan | |

**Recommendation:** <one sentence + conditions: pin exact version, disable scripts, watch with postmortem watch>
```

## Rules

- `--package-lock-only` resolution (no scripts executed) is the hard boundary; if an ecosystem cannot resolve without executing, inspect the registry metadata instead and say the check was shallower.
- "Popular" is not "safe" and "small" is not "risky"; the verdict argues from evidence, not stars.
- Vet the exact version being adopted; a clean 4.2.1 says nothing about 4.3.0-beta.
