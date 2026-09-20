---
name: smishing-campaign
description: Analyze a batch of reported SMS/text messages through mlab.sh - score each for smishing, cluster them into campaigns, and extract the shared infrastructure (URLs, callback numbers, sender patterns). Use when the user has multiple reported SMS from users/employees, or investigates an SMS phishing wave. For a single message, phishing-triage does the job.
---

# Smishing Campaign Analysis
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Workflow

### 1. Score every message

Run `smishing_risk` on each message body in RAW form - the IOC extractor skips defanged text, so refang first if reports arrived defanged. Pass `country` matching the fleet's locale (default `fr`; ask if the batch is mixed). Keep per-message: score, band, filter action, reasons, extracted IOCs. The tool is offline and quota-free, so volume is not a problem.

### 2. Cluster into campaigns

Group messages by shared evidence, in order of strength:
1. Same URL, domain, or URL pattern (same registrar-fresh domain family counts)
2. Same callback phone number
3. Same brand impersonated + same pretext (package delivery, tax refund, bank alert)
4. Near-identical wording

One campaign = one cluster. Singletons stay singletons; do not force them into clusters.

### 3. Enrich each campaign's infrastructure once

Per campaign (not per message - this is where batch discipline saves quota):
- Distinct URLs → `scan_url`; a shortener central to the campaign may justify `resolve: true`, announced.
- Callback numbers → `scan_phone` (VoIP + country/operator mismatch with the impersonated brand = strong signal).
- Central domains → offer `start_domain_scan` (quota; check `get_scan_limits` for several).
- Sender emails, wallets in the messages → `scan_email` / `scan_crypto`.

### 4. Report

```
## SMS wave: <N> messages -> <K> campaigns

| Campaign | Msgs | Brand/pretext | Score range | Action |
|---|---|---|---|---|

### Campaign <k>: <brand/pretext>
- Sample message (1, verbatim, defanged)
- Infrastructure: <URLs, numbers, domains + verdicts>
- Filter recommendation: <allow/quarantine/drop + the reasons the scorer gave>

### Blocklist (defanged, ready to deploy)
<distinct URLs, domains, numbers across all campaigns>

### False-positive review
<messages scored high that look legitimate on inspection - flag, never bury>
```

## Rules

- Defang every IOC in the report (`hxxp`, `[.]`); the report itself gets forwarded and must not carry live links.
- Quote the scorer's weighted reasons per campaign; a bare score convinces nobody.
- Never call, reply to, or open anything from the messages to verify; scoring is static.
- A `clean`-band message reported by a user still gets a line in the false-positive review - users report things for reasons.
