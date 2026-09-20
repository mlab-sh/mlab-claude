---
name: license-check
description: Inventory the licenses of a project's dependency graph and enforce a policy over them using postmortem - denied licenses, unknown licenses, dual-licensing. Use when the user asks about license compliance, what licenses their dependencies carry, whether GPL/AGPL is present, or wants a license gate for CI.
---

# License Compliance Check

> CLI details and preflight: `${CLAUDE_PLUGIN_ROOT}/docs/postmortem-reference.md`.

## Workflow

### 1. Preflight

`postmortem --version`. Ask for the policy if none is stated and none exists in `postmortem.conf` (auto-loaded `[license]` section): which SPDX ids are forbidden (typical: AGPL-3.0-only and copyleft for proprietary SaaS), and does unknown fail?

### 2. Inventory

```
postmortem licenses <path> --online --packages --json -o -
```

`--online` is usually needed: only npm and composer declare licenses offline. `--unknown-only` for a focused pass on the unresolved set.

### 3. Enforce

Either via flags: `--deny <SPDX>` (repeatable), `--allow <SPDX>` (allowlist mode), `--fail-on-unknown` (pair with `--online`); or via the project's `postmortem.conf`. Key semantics to relay: a dual-licensed package fails only when EVERY alternative it offers is denied - report the alternative that saves it.

### 4. Report

```
## License inventory: <project> - <compliant | violations | unknowns to resolve>

| License | Packages | Policy |
|---|---|---|

### Violations
<package, license, why denied, the dependency path that pulls it in (postmortem why <pkg> <path>)>

### Unknown licenses
<the actionable set: package + where to check>

### CI gate
<the exact licenses command with policy flags, or the postmortem.conf [license] block to commit>
```

## Rules

- You report what the graph declares; whether a license is acceptable for the user's product is their counsel's call - state findings, not legal advice.
- Unknown is not a violation; it is unresolved. Keep the two lists separate.
- Dev-only dependencies often deserve a softer policy: mention `--omit dev` when a violation sits in the dev tree.
