---
name: repo-ioc-sweep
description: Inventory what a codebase talks to - extract embedded URLs, IPs, domains, wallets, base64 blobs and shell scripts from a repo, then enrich each endpoint through mlab.sh. Use when the user asks what a repo connects to, wants an egress inventory, or vets a vendor/third-party codebase for suspicious embedded infrastructure.
---

# Repo IOC Sweep

> mlab output shapes: `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`.

"This repo talks to whom?" Answer with an inventory, then a verdict per endpoint. Dependency CODE is supply-chain-audit's job; this skill sweeps the repo's own files.

## Workflow

### 1. Extract

Sweep tracked text files (skip vendored deps, lockfiles, node_modules, minified bundles, test fixtures unless asked) with grep/ripgrep for:
- URLs and bare domains (config files, source, CI workflows, Dockerfiles, install docs)
- IP literals (both in code and in configs - an IP literal in source is a finding by itself)
- crypto wallet shapes (BTC/EVM/SOL patterns)
- long base64/hex blobs (decode ONE layer locally, never execute; re-sweep the decoded content)
- webhooks, telemetry endpoints, update URLs, package registries beyond the defaults

Shell scripts in the repo (install.sh, CI steps, hooks) → `scan_bash` each: it flags dangerous constructs and extracts IOCs for you.

### 2. Classify before enriching

Dedupe, then bucket: documentation-only (README badges, examples) / build-time (CI, registries) / runtime (code paths that will fire in production). Runtime endpoints matter most; say which bucket each finding sits in.

### 3. Enrich

`detect_ioc` on ambiguous strings; `scan_url` per URL; `scan_ip` on IP literals (quota - `get_scan_limits` first on large sets, budget like ioc-batch); `scan_crypto` on wallets (crypto quota, chain rules); domain scans only for the few domains central to the verdict.

### 4. Report

```
## Egress inventory: <repo>

**Verdict:** <expected profile | surprises found | suspicious>

| Endpoint | Where (file:line) | Bucket | Verdict | Evidence |

### Surprises
<anything a user of this software would not expect it to contact - with the code path>

### Wallets / blobs
<decoded layers, what they contained>

### Blocklist candidates
<defanged>
```

## Rules

- Every finding carries file:line; an inventory without locations cannot be acted on.
- An endpoint being unknown to mlab is not a finding; an endpoint UNEXPECTED for this software is - argue from purpose.
- Decode blobs statically only; never run repo code to observe its traffic.
