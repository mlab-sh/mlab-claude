---
max_turns: 20
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: The ioc-triage skill fires, scan_ip runs via the mocked server, and the reply is a structured verdict identifying a TOR exit node.
---

hey, our EDR flagged an outbound connection to 185.220.101.1 from a build server. can you check what this IP is?
