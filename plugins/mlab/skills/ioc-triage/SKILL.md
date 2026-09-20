---
name: ioc-triage
description: Triage any indicator of compromise (IOC) through mlab.sh - IPs, CIDR ranges, domains, URLs, file hashes, email addresses, MAC addresses, phone numbers or blockchain addresses. Detects the indicator type, runs the right lookups, pivots to malware families, threat actors and CVEs, and produces an analyst-ready verdict. Use when the user pastes one or more indicators, or asks to check, triage, enrich, or investigate an IOC.
---

# IOC Triage
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


Turn one or more raw indicators into a structured, evidence-backed verdict using the mlab.sh tools.

## Workflow

### 1. Type every indicator

Run `detect_ioc` on each ambiguous indicator to identify its type. Exception: an obvious IP goes straight to `scan_ip` - `detect_ioc` on an IP performs the full IP lookup itself and consumes IP quota, so typing it first pays twice. For everything else `detect_ioc` is free and also returns immediate context (hash algorithm for digests, type for domains).

If the user gave more than ~5 indicators that will consume scan quota (IPs, domains), call `get_scan_limits` first and tell the user how much quota the batch will use before proceeding.

### 2. Run the type-specific lookup

| Type | Tool | Notes |
|------|------|-------|
| IPv4 / IPv6 | `scan_ip` | Geo, ASN, datacenter/proxy/VPN/TOR flags, rDNS, RDAP + abuse contact. Uses quota. |
| CIDR range | `scan_ip` | Pure arithmetic, no quota. |
| Domain | `start_domain_scan` then poll `get_domain_scan_results` until `status` is `completed` | DNS, subdomains, SSL certs, security.txt. Uses quota. |
| URL | `scan_url` | Static analysis by default. Only set `resolve: true` to unmask a shortener, and say so: it makes an outbound request to attacker-controlled infrastructure. |
| File hash | `scan_hash` | Multi-feed verdict. Check `sources_answered` before treating `unknown` as meaningful; `unknown` is NOT `clean`. If `mlab_sample` is present, cite the full mlab report link. |
| Email | `scan_email` | Quote the weighted `reasons`, not just the 0-100 score, and check `score.conclusive`. |
| MAC | `scan_mac` | Offline OUI lookup. |
| Phone | `scan_phone` | E.164 form preferred. |
| Crypto address | `scan_crypto` | Uses crypto quota. For EVM addresses the chain CANNOT be derived from the address: ask the user which chain, or pass the one given in context. An empty result on a guessed chain is not a clean verdict. |

### 3. Pivot for attribution

- Hash verdict names a malware family → `search_actors` with the family name, then `get_actor` on the best match for aliases, TTPs and exploited CVEs.
- Any CVE surfaces → `cve_detail` for CVSS/EPSS/KEV, then `actors_by_cve` for attribution.
- URL analysis reveals a host → treat the host as a new domain indicator (step 2) if the user wants depth.

Stop pivoting after two hops unless the user asks for a deeper investigation; note the untaken pivots in the report instead.

### 4. Report

Produce a compact report in this shape:

```
## Verdict: <malicious | suspicious | inconclusive | likely benign>

| Indicator | Type | Verdict | Key evidence |
|---|---|---|---|

### Evidence
<per-indicator findings, each tied to the tool result that produced it>

### Attribution
<actors / families / CVEs, with confidence and the source link>

### Recommended actions
<block / hunt / escalate, concrete and prioritized>

### Not checked
<pivots deliberately skipped, quota constraints>
```

## Rules

- Never invent a verdict a tool did not return. `unknown` and empty results stay `inconclusive`.
- Always distinguish "no feed knows this hash" from "this hash is clean".
- Cite quota usage when the user runs batches.
- If a scan fails or quota is exhausted, say so plainly and report on what completed.
