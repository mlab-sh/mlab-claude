# postmortem CLI reference - verified against v2.4.0

[postmortem](https://github.com/mlab-sh/postmortem) is mlab's supply-chain scanner: a single static Rust binary, no telemetry, offline by default - the network is touched only by `--online` (source-repo reputation), `--vulns` (advisories via vuln.mlab.sh) and `--image` pulls.

## Preflight (every postmortem-based skill starts here)

Run `postmortem --version`. If missing, do NOT install it silently; tell the user and offer:
- macOS: `brew install mlab-sh/tap/postmortem`
- Debian/Ubuntu: signed apt repository (see the repo README)
- Any platform: prebuilt binaries (x86_64/arm64, macOS/Linux/Windows) from GitHub releases, or `cargo install` from source
Then re-check. If the user declines, fall back to what the mlab MCP tools alone can do (`scan_sbom` on a lockfile) and say what is lost (no local malware scan, no typosquat/maintainer analysis).

## Command map (what answers what)

| Question | Command |
|---|---|
| Is there malicious code in my deps? | `scan <path>` - fully offline static analysis: IOCs, obfuscation, install hooks, sensitive APIs |
| Overall health, graded | `audit <path> [--online] [--vulns]` - one-shot: malware + inventory + reputation + vulns |
| What does this PR / change introduce? | `diff <pr-url>` or `diff <old> <new> [--online] [--vulns]` - cost scales with the diff |
| What depends on what? | `tree <path> [--depth N] [--online] [--vulns]` |
| Who controls my tree? | `tree <path> --online --human` - maintainer graph by compromise reach |
| Why is X installed / what if X is compromised? | `why <pkg> <path>` / `why <pkg> <path> --blast` |
| Which deps run code at install? | `scripts <path>` |
| Package history: handovers, install-script appearance, repo moves | `timeline <pkg> [path]` - **npm only** |
| SBOM | `sbom <path> -o <file> [--online] [--omit dev]` - CycloneDX 1.5 JSON |
| Licenses + policy | `licenses <path> [--online] [--packages] [--unknown-only] [--deny SPDX] [--allow SPDX] [--fail-on-unknown]` |
| Container image instead of a directory | `--image <ref>` on scan/tree/audit/sbom (flattened via docker, never run; `--layers` for layer attribution and deleted-credential checks) |

Ecosystems: Node, Python, Rust, Ruby, PHP, Go, Java. `system` audits OS package managers.

## Output modes and gotchas

- `--json` everywhere; `-o -` forces stdout (without `-o`, scan/sbom write a timestamped file in cwd - prefer explicit `-o`).
- `scan` also does `--html` and `--sarif` (SARIF 2.1.0 for GitHub Code Scanning); machine formats require a single path.
- `tree --json` with multiple targets needs `--allow-multiple` and CHANGES THE SHAPE to an array.
- `scan --severity <level>` sets the CI gate: minimum severity causing non-zero exit (default high). Non-zero exit codes are gate semantics, not tool failure - check stderr before concluding the run broke.
- `scan --json` top-level shape: `{schema_version, root, ecosystems[], dependencies[], findings[]}`.
- `audit` verdicts observed: CLEAN, and CRITICAL when a high-severity vuln is present ("worst: high" wording). Grades come with counts per category (malware / reputation / vulns), each "not checked" until its flag is passed.
- `audit --allow-test-files` includes IOC findings in test/fixture dirs (excluded by default - remember this when a "clean" verdict surprises you on a repo full of samples).
- `--omit dev --omit optional`: a package is dropped only when EVERY path to it goes through an omitted edge; Go is unaffected (no scope info).
- `diff` accepts a GitHub PR URL directly as `<OLD>` (both sides come from the PR, `<NEW>` omitted).
- `postmortem.conf` in the project dir is auto-loaded for `[license]` policy.

## Division of labor with the mlab MCP tools

postmortem = local detection (code-level, offline). mlab MCP = cloud intelligence (attribution, reputation, vuln detail). The hybrid pattern every skill follows: run postmortem locally, take its findings (hashes, URLs, IPs, package names, families), pivot each through the MCP tools (`scan_hash`, `scan_url`, `scan_ip`, `search_actors`, `cve_detail`...), and merge into one report. `--vulns` already calls vuln.mlab.sh server-side, so do not re-run `scan_sbom` on a project you just audited with `--vulns` - same data.
