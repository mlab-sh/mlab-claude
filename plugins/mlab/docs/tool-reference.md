# mlab.sh tool reference - verified field notes

Everything below was verified against the live API (2026-09). Read this before interpreting any tool result; the gotchas here override intuition.

## Quotas (get_scan_limits)

Four per-day pools, shared org-wide: `ip`, `domain`, `file`, `crypto`.

| Consumes quota | Free |
|---|---|
| `scan_ip` (address) | `scan_ip` (CIDR range) |
| `detect_ioc` **on an IP** (it performs the full IP lookup) | `detect_ioc` on other types |
| `start_domain_scan` (unless a recent scan is returned) | `scan_hash`, `scan_url`, `scan_email` |
| `scan_crypto` (crypto pool) | `scan_bash`, `smishing_risk`, `scan_phone`, `scan_mac` |

Consequence for batches: type IPs by shape locally and go straight to `scan_ip`; reserve `detect_ioc` for genuinely ambiguous strings, or you pay twice.

## detect_ioc

Returns `type` plus immediate context. For IPs the result is already a geolocation/reputation lookup (flat fields: `as`, `isp`, `org`, `country`, `lat/lon`). For domains it types without scanning. For hashes it identifies the algorithm only.

## scan_ip

Key blocks in the result:
- `tor`: `is_tor`, `is_exit`, `is_guard`, `fingerprint`, `nickname`, `first_seen`/`last_seen` (epoch). Consensus-based, authoritative.
- `rdns`: `name`, `forward_confirmed` (true = the rDNS name resolves back to this IP; an unconfirmed rDNS is decorative and spoofable).
- `rdap`: `abuse_email` / `abuse_name` (use in reports for the abuse-reporting recommendation), `cidr`, `holder`, `registered`.
- Flags: `hosting`, `proxy`, `mobile`.
- `ikwyd`: BitTorrent DHT observation data for the IP (`observations`, `torrents`, `categories`). Signal about what the host is used for, not maliciousness.

## scan_hash

- `verdict`: `known_malicious` / known-good / `unknown` / `unavailable`. Check `sources_answered` vs `sources_queried` before treating a miss as meaningful; individual sources report `status`: `hit` / `miss` / `disabled` (a disabled feed answered nothing - do not count it as a miss).
- CIRCL hashlookup is nominally the known-GOOD feed but carries `malicious: true` entries too (EICAR sits in NSRL flagged malicious, `trust: 100`). Read per-source `malicious`, not the feed's reputation.
- `mlab_sample.known`: whether mlab itself analysed this exact file - independent of all external feeds, comes with a `note` and report link when known.
- `family` is filled when a bad-feed names one: that string is your `search_actors` pivot.

## scan_url

- `findings[]`: `{title, severity, detail}` - severities observed: low/medium/high.
- `host_context.scanned`: false means mlab has no completed domain scan; the accompanying `note` says nothing is known either way. Registration age only appears when a domain scan exists.
- `resolution`: null until you call with `resolve: true`.

## scan_email

- `score`: `{value 0-100, band, conclusive, reasons[]}` where each reason is `{code, label, weight}` (negative weights lower risk, e.g. `email.dmarc_enforced: -10`). Quote reasons, never the bare number.
- `findings[]`: human-readable `{title, severity, detail}`.
- `domain_scan.mail_source`: `live` (resolved over DNS now) vs from a stored scan. `spoofability.verdict` + `summary` are pre-written analyst language - reuse them.
- `mailbox_type`: e.g. `Disposable`; booleans `is_disposable`, `is_role`, `is_free_provider`.

## smishing_risk

- Result: `{score, band, action, reasons[], iocs[], mode}`; reason labels are localized by `country` pack (fr labels for `fr`).
- **The IOC extractor does not parse defanged text.** `hxxps://x[.]y` in the message yields `iocs: []`. Submit the original raw message; if you only have a defanged copy, refang before scoring, then re-defang for the report.
- Bands observed: clean / suspect / high / critical; actions: allow / quarantine / drop.

## scan_bash

- `suspicious[]`: `{label, hint, line, snippet}` - quote snippet+line directly in reports.
- `iocs`: grouped by type (`IPv4`, `URL`, ...), each `{value, link}` where link is a relative mlab platform path.
- `file.sha256` is computed for you: feed it to `scan_hash` to check if the exact script is already known.

## scan_crypto

- `chain_ambiguous` + `chain_source` (`explicit` when you passed it) + `chain_candidates` tell you how trustworthy the chain attribution is.
- `address_info.checksum`: `invalid` = malformed/typo, stop there.
- `intel`: `{labels[], categories[], risk_score, risk_level, sanctions.is_sanctioned}`. Coverage is not omniscient: an empty `labels` with `risk_score: 0` means no adverse data in the sources, NOT a clean bill - notorious addresses can come back unlabeled. Corroborate before writing "clean".

## Domain scans

- `start_domain_scan` returns `{status: "completed", results: {...}}` immediately when a recent scan exists - handle both shapes (immediate results vs `{status: "started"}` then poll `get_domain_scan_results`).
- `results` structure: `dns.resolve[]` (per-host a/aaaa/cname), `dns.txt` (`spf`, `dmarc`, `dkim` selectors, `raw[]`), `files.robots_txt` (full text) / `files.security_txt` (`"not found"` as a string), `ssl[]` (can be empty), `subdomains[]`, `subdomains_suspicious[]` (`{subdomain, keyword}` - keyword-based flags like `prod`, `db`; treat as hints, not findings).

## CVE tools

- `cve_search` list entries ALREADY carry `epss_score`, `in_kev`, `cvss_score`, `cvss_severity`: KEV/EPSS triage happens on search results, no `cve_detail` fan-out needed. `total_results` + `start_index` for pagination.
- `cve_detail` adds: `cvss_vector` + `cvss_breakdown[]`, `epss_percentile`, `kev_date_added` / `kev_due_date` (the CISA remediation deadline - useful pressure in reports), `weaknesses[]` (CWE), `references[]` with tags.

## Actor tools

- Slugs are compound and unguessable: Lazarus is `lazarus-group-hidden-cobra-labyrinth-chollima`, `primary_name` is "Lazarus Group, Hidden Cobra, Labyrinth Chollima". ALWAYS resolve via `search_actors` first; never construct a slug.
- `search_actors` matches primary names only (compound names make many aliases hit anyway, but Microsoft/Mandiant-style names like "Zinc" or "UNC4899" will not - they live in the full record's `aliases[]`).
- `get_actor` returns: `actor.description` (contains `{{...}}` cross-reference markup - strip or rewrite it), `actor.tlp` (respect it), `aliases[]` with per-vendor `source`, `cves[]`, `techniques[]` (MITRE ATT&CK `mitre_id`/`name`/`tactic` - use for the tradecraft section and hunting angles), `tools[]` (`kind`: malware/tool, MITRE S-ids), `references[]`.
- `actors_by_cve` can return pseudo-actors like `[Unnamed groups: China]` (slug `unnamed-groups-china`): report as "unattributed activity linked to <origin>", not as a named group.

## History & bookmarks

- `get_scan_history` logs every tool type (`ip`, `domain`, `file`, `hash`, `url`, `email`, `sms`, `phone`, `mac`, `crypto`, `bash`), though its `type` FILTER only accepts ip/domain/file. Useful to see what teammates scanned (org-shared).
- `get_bookmarks` entries carry `{value, type, time}`; only ip/domain/file(hash) types exist.
