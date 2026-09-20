---
name: domain-surface
description: Review the external attack surface of a domain via mlab.sh - DNS records, subdomains, SSL certificates, mail spoofability (SPF/DKIM/DMARC), security.txt and exposed hosts. Use when the user asks to scan/audit/review a domain, check what a company exposes, verify their own perimeter, or assess a third party's external posture.
---

# Domain Surface Review
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Workflow

### 1. Scan the domain

`start_domain_scan` on the apex domain. Domain scans consume quota; for a list of domains, check `get_scan_limits` first. If a recent scan exists you get results immediately; otherwise poll `get_domain_scan_results` every few seconds until `status` is `completed`. Tell the user the scan is running rather than going silent.

### 2. Walk the surface

From the completed scan:

- **Subdomains**: group by function (mail, dev/staging, vpn, api, cdn...). dev-, staging-, test-, old- prefixes on public DNS are findings in themselves.
- **DNS records**: dangling records (CNAME to unclaimed services = takeover candidates), wildcard entries, suspicious TXT records.
- **SSL certificates**: expired or expiring soon, weak configs, certs revealing internal hostnames.
- **security.txt / robots.txt**: present? robots.txt disclosing sensitive paths?

### 3. Deepen selectively

- Interesting exposed hosts → `scan_ip` on their A records: datacenter vs corporate ASN, proxy/VPN flags, rDNS coherence. Each lookup consumes quota; pick the hosts that matter, do not carpet-scan.
- Mail posture → `scan_email` on `postmaster@<domain>`: returns the domain's SPF/DKIM/DMARC spoofability verdict. A spoofable domain is usually the report's top finding for anyone doing phishing-awareness.

### 4. Report

```
## External surface: <domain>

**Posture:** <tight | average | leaky> - <one sentence>

### Top findings
<3-5 prioritized, each: finding -> risk -> fix>

### Surface inventory
| Subdomain | Points to | Notes |

### Mail spoofability
<SPF/DKIM/DMARC verdict + what it permits an attacker>

### Certificates
<expiries, anomalies>

### Not covered
<what a DNS-based scan cannot see: closed ports, appl-level vulns, internal segmentation>
```

## Rules

- This is reconnaissance from public data, not a pentest; the report says so and never claims exploitability.
- For a third-party domain, stay descriptive. Do not suggest intrusive follow-ups against infrastructure the user does not own.
- Takeover candidates are reported as "candidate, verify ownership of <service>" - confirming one means claiming the resource, which is not this skill's job.
