---
name: intel-report
description: Turn a finished investigation into a client-ready CTI report - TLP marking, executive summary, findings with confidence levels, defanged IOC annex. Pure writing playbook on top of results already gathered. Use when the user asks for a report, deliverable, write-up or summary of an investigation done in this or a previous session.
---

# CTI Report Writing
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


This skill produces the document; the investigation is already done. If material is missing, run the relevant skill first (ioc-triage, script-triage...) - a report is never padded to hide a gap, the gap is stated.

## Gather

Source material, in order: this conversation's findings; the case file (`.mlab/cases/`, see case-notes); `get_bookmarks` and `get_scan_history` for platform-side state. Ask the user for: audience (exec / SOC / client), TLP marking, and whether attribution may be named in writing.

## Structure

```
# <Title: what happened, not "Security Report">
TLP:<CLEAR|GREEN|AMBER|AMBER+STRICT|RED> | <date> | <author/org>

## Executive summary
<5 sentences max, no jargon: what happened, impact, what we did, what remains.
 Written LAST, placed first.>

## Key findings
<numbered; each: claim -> evidence -> confidence (high/medium/low)>

## Timeline
<dated, factual, UTC unless the client says otherwise>

## Technical analysis
<the how: per finding, with tool evidence cited. Screenshots/links to mlab reports where they exist>

## Attribution
<"attributed to X with <confidence>, based on <evidence>" - or "attribution not established".
 Never stronger than the evidence; overlaps ("consistent with") are not identity.>

## Recommendations
<prioritized, actionable, owner-assignable. No boilerplate ("raise awareness").>

## Annex A - IOCs
<table: indicator (DEFANGED) | type | role | first seen | verdict source.
 Machine-usable: also offer the CSV.>
```

## Confidence language

Calibrate and keep it consistent:
- **High**: multiple independent sources agree (feed verdict + behavior + infrastructure).
- **Medium**: single solid source, or converging weak signals.
- **Low**: plausible, evidence thin - say what would raise it.

## Rules

- Every claim traces to evidence gathered with the tools; nothing enters the report from memory or general knowledge without being labeled as context.
- IOCs are defanged everywhere in the document, including inline mentions.
- The executive summary contains zero IOCs, zero CVE numbers, zero tool names.
- TLP marking appears on the first line and in the footer; if the user gave none, default TLP:AMBER and say so.
- Deliver as a file (markdown, or the format the user asks). Long reports: outline first, validate, then write.
