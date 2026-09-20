---
name: phishing-triage
description: Analyze a suspicious email, SMS or message for phishing/smishing using mlab.sh - sender address reputation, domain spoofability (SPF/DKIM/DMARC), embedded URL analysis, callback phone numbers and smishing scoring. Use when the user pastes a suspicious message, reports a phishing attempt, or asks whether an email/SMS/link is legit.
---

# Phishing / Smishing Triage
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Workflow

Decompose the message into its indicators, score each, then give one combined verdict.

### 1. The message body (SMS or short message)

For an SMS or text-style message, run `smishing_risk` on the full RAW body - the IOC extractor does not parse defanged text, so refang (`hxxp` -> `http`, `[.]` -> `.`) if you only have a defanged copy. Pass `country` matching the user's locale pack (default `fr`). Use the returned `iocs[]` as your indicator list rather than parsing by hand, and re-defang everything for the report.

### 2. The sender

- Email address → `scan_email`. Key signals: disposable/role mailbox type, and the domain's spoofability verdict (SPF/DKIM/DMARC). A spoofable domain means the From header proves nothing.
- Phone number → `scan_phone`. VoIP line type and country/operator mismatches with the claimed brand are strong signals.

### 3. Embedded links

For each URL, run `scan_url` (static). Highlights to check: punycode/Unicode host confusion, credentials-in-URL, IP-literal hosts, dangerous file extensions, known shorteners, embedded redirect targets, and the host's registration age when a previous domain scan exists.

Only set `resolve: true` on a shortener when the destination is needed for the verdict, and note in the report that the link was followed. If the sender domain itself looks central to the case and is unscanned, offer to run `start_domain_scan` (it consumes quota).

### 4. Verdict

```
## Verdict: <phishing | suspicious | likely legitimate> (confidence: <low/med/high>)

**One-line summary for the end user:** <plain-language, no jargon>

### Signals
| Element | Finding | Weight |
<sender, each link, body score - with the tool-reported reasons>

### What to do
<for an end user: don't click / report to X / delete. For a SOC: block indicators, search mail logs for the sender/URL pattern>

### Extracted IOCs
<clean list, ready to paste into a blocklist or hunt>
```

## Rules

- Quote the scoring reasons the tools return; never present a bare number as the argument.
- A low smishing score with `conclusive: false` is not a clean bill; say what could not be checked.
- Brand impersonation is a judgment call: state which brand the message imitates and which signals support that.
- Never advise the user to interact with the message (reply, call the number, open the link) to "test" it.
