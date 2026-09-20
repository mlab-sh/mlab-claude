---
name: sbom-export
description: Generate a CycloneDX SBOM locally with postmortem and push it through mlab.sh vulnerability scanning - for projects or container images where no single lockfile can be pasted. Use when the user asks for an SBOM, needs vuln scanning of a container image or a multi-ecosystem repo, or when sbom-audit lacks a usable lockfile.
---

# SBOM Export + Scan

> CLI details and preflight: `${CLAUDE_PLUGIN_ROOT}/docs/postmortem-reference.md`. mlab output shapes: `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`.

sbom-audit takes a lockfile the user pastes. This skill covers the rest: multi-ecosystem repos, container images, "give me an SBOM" as a deliverable.

## Workflow

### 1. Preflight

`postmortem --version`. Target: project directory or `--image <ref>` (image is flattened, never run; needs docker present).

### 2. Export

```
postmortem sbom <path> -o sbom.cdx.json [--online] [--omit dev]
```

Always pass `-o` explicitly (default writes a timestamped file). `--online` fills licenses the lockfile does not record. `--omit dev` when the SBOM describes what ships. For an image: `postmortem sbom --image <ref> -o sbom.cdx.json`.

### 3. Scan

Read `sbom.cdx.json` and pass its content to `scan_sbom` with `format: "cyclonedx"`. Then the standard ladder: KEV → EPSS → CVSS, `cve_detail` on the shortlist, `actors_by_cve` on KEV entries. (If the user instead runs `postmortem audit --vulns`, that already queries vuln.mlab.sh - do not double-scan; use this path when the SBOM file itself is the deliverable.)

### 4. Deliver

Two artifacts: the SBOM file itself (the user keeps it - name, where it was written, component count) and the vuln report in sbom-audit's exact format (P0 KEV / P1 / P2 / remediation plan).

## Rules

- The SBOM states what the lockfiles resolve, not what is deployed; for the deployed truth, prefer `--image` against the production image.
- Component count sanity check: an SBOM with suspiciously few components usually means an ecosystem was not detected - list which ecosystems postmortem found and compare with what the repo contains.
- Never edit the generated SBOM by hand; regenerate.
