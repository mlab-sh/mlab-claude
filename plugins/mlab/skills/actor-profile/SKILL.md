---
name: actor-profile
description: Build a threat actor profile brief from mlab.sh actor intelligence - origin, motivations, targeting, aliases, exploited CVEs, tools and techniques. Use when the user names a threat actor or APT group (APT28, Lazarus, Scattered Spider...), asks who is behind a campaign, or wants to know whether an actor targets their sector or exploits a given CVE.
---

# Threat Actor Profile
> Before interpreting any tool result, read `${CLAUDE_PLUGIN_ROOT}/docs/tool-reference.md`: verified output shapes and gotchas that override intuition.


## Workflow

### 1. Resolve the actor

Call `search_actors` with the name the user gave. Slugs are compound and unguessable (Lazarus is `lazarus-group-hidden-cobra-labyrinth-chollima`), so never construct one - always resolve via search. The search matches primary names only, NOT aliases; primary names are themselves compound ("Lazarus Group, Hidden Cobra, Labyrinth Chollima") so common names often hit anyway, but vendor codenames ("Zinc", "UNC4899") will not - retry with the best-known primary name. Optionally narrow with `origin`, `motivation` or `sector` when the user's question implies them ("Chinese actors targeting finance").

Take the `slug` from the best match and call `get_actor` for the full record.

If several actors plausibly match, list them with one line each and ask which one, unless context makes it obvious.

### 2. Enrich the CVE angle

`get_actor` returns the CVEs attributed to the actor. For the most relevant ones (recent, or matching the user's stack if known), call `cve_detail` to get CVSS, EPSS and KEV status. If the user came in from a CVE ("who exploits CVE-2023-2868?"), start from `actors_by_cve` instead and profile each returned actor briefly.

### 3. Brief

```
## <Primary name> (<slug>)
Also known as: <aliases>

**Origin:** <suspected origin> | **Motivation:** <motivations> | **Active since:** <first seen>
**Targets:** <sectors> in <countries>

### Summary
<2-3 sentences from the profile description, in your own words>

### Tradecraft
<`techniques[]` grouped by ATT&CK tactic with MITRE IDs; `tools[]` split malware vs tools with S-ids>

### Exploited vulnerabilities
| CVE | Product | CVSS | KEV | Why it matters |

### Relevance to you
<only if the user's sector, stack or geography is known from context: a concrete read on exposure>

### Sources
<external references from the record>
```

## Rules

- Attribution is always "suspected"/"attributed", never stated as certain fact. Respect the record's `tlp` marking.
- `actors_by_cve` can return pseudo-actors like "[Unnamed groups: China]": report those as "unattributed activity linked to <origin>", not as a named group.
- The `description` field contains `{{...}}` cross-reference markup: rewrite it, never paste it raw.
- Do not pad the profile with general knowledge about the actor from memory; the brief reflects the mlab record, and anything added from elsewhere is labeled as such.
- When comparing actors or answering "who targets X", quote the fields (sectors, motivations) rather than inferring.
