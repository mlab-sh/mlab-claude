---
name: crypto-trace
description: Triage blockchain addresses through mlab.sh - labels, sanctions status, risk scoring and address type across Bitcoin, Ethereum and 12 other EVM chains, Tron, Solana, TON and Dogecoin. Use when the user provides a crypto/wallet address, investigates a ransomware payment or scam wallet, or needs a sanctions/AML check on an address.
---

# Crypto Address Triage
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Workflow

### 1. Establish the chain BEFORE scanning

This is the step everyone gets wrong. All 13 EVM chains share one address format, so the chain cannot be derived from an EVM address:

- Non-EVM (BTC, SOL, TRX, TON, DOGE): the family is decoded from the address, proceed directly.
- EVM (0x...): ask the user which chain, or take it from context (the tx explorer link, the dApp mentioned). If genuinely unknown, scan the likely chains one by one (ETH, then BSC, TRON-bridged cases aside) and label each result with its chain.

Never present an empty result on a guessed chain as "no findings". When `chain_ambiguous` is true in the result, say so explicitly.

### 2. Scan

`scan_crypto` with `address` and explicit `chain` (each lookup consumes the org's daily crypto quota; `get_scan_limits` before scanning a list). Check `address_info.checksum` first: `invalid` means a malformed address, usually a typo in the source - report that and stop, everything else would be noise.

### 3. Read the result

Report in this order of importance:
1. **Sanctions status** - a sanctioned address is a compliance stop, whatever the rest says.
2. **Labels** - exchange, mixer, scam, ransomware, bridge... with their source.
3. **Risk score and address type** (EOA/contract, age, activity shape).

### 4. Pivot

Addresses often arrive inside something else: a smishing SMS, a ransom note, a script. Offer to triage the carrier (smishing-campaign, script-triage) and any co-occurring IOCs. Bookmark confirmed-bad addresses' related infrastructure (IPs/domains) with `add_bookmark` if a case is open.

### 5. Report

```
## <address> (<chain>)

**Verdict:** <sanctioned | high-risk | labeled: X | no adverse findings on <chain>>

| Check | Result |
|---|---|
| Checksum | |
| Sanctions | |
| Labels | |
| Risk score | |
| Type | |

### Interpretation
<what this means for the user's actual question: pay/don't pay, report to X, file SAR...>

### Chains not checked
<for EVM: every chain not scanned, stated plainly>
```

## Rules

- "No labels" on one chain proves nothing about the other 12; the report always says which chains were checked.
- Empty `labels` with `risk_score: 0` means no adverse data in the sources, not a clean address - notorious addresses can come back unlabeled. Say "no adverse findings in the queried sources", never "clean".
- Sanctions findings are stated verbatim with source; the user's compliance obligations are theirs to interpret - point to the finding, not to legal conclusions.
- Never estimate balances, flows or counterparties: the tool does not return them, so the report does not contain them.
