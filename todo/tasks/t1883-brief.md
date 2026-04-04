# t1883: Private Detection Repo for Attribution Monitoring

ref:GH#17323

## Session Origin

Interactive session (t1880 attribution protection work). The user correctly identified that putting detection search patterns in the public aidevops repo would reveal what strings we're looking for, allowing a copycat to strip them first.

## What

Create a separate **private** GitHub repo (`marcusquinn/aidevops-provenance` or similar) containing:

1. **GitHub Code Search script** — periodically searches for distinctive strings from the aidevops codebase across all public GitHub repos
2. **Canary dashboard** — reads the Cloudflare KV store (`provenance-ping` Worker) and presents a summary of non-canonical origin pings
3. **Attribution manifest** — the private inventory of all watermark artifacts, their locations, and verification procedures (currently at `~/.aidevops/.agent-workspace/work/provenance-worker/MANIFEST.md`)
4. **Scheduled routine** — launchd plist or GitHub Actions cron to run detection periodically

## Why

The public aidevops repo cannot contain detection logic without revealing what we're searching for. A copycat monitoring our repo would see the search strings and strip them preemptively. The detection infrastructure must be in a separate private repository.

## How

### Repository Structure

```
aidevops-provenance/
├── MANIFEST.md              # Attribution artifact inventory (moved from agent workspace)
├── scripts/
│   ├── search-github.sh     # GitHub Code Search for distinctive strings
│   ├── read-canary-kv.sh    # Read Cloudflare KV pings via API
│   └── generate-report.sh   # Combine search + canary into summary report
├── config/
│   ├── search-strings.json  # Distinctive strings to search for (PRIVATE)
│   └── known-forks.json     # Known legitimate forks to exclude from alerts
├── reports/                  # Generated reports (gitignored or auto-committed)
├── worker/                   # Cloudflare Worker source (backup copy)
│   ├── wrangler.toml
│   └── src/worker.js
└── .github/
    └── workflows/
        └── detect.yml        # Weekly cron job for detection
```

### Search Strings Strategy

The `search-strings.json` should contain strings that are:
- **Unique to aidevops** — not found in other projects naturally
- **Functional** — embedded in code logic, not just comments (harder to strip)
- **Varied** — mix of error messages, function names, variable patterns, comment phrases

Categories to search for:
1. SPDX copyright text: `"SPDX-FileCopyrightText: 2025-2026 Marcus Quinn"`
2. Canary endpoint: `"provenance-ping.marcusquinn.workers.dev"` or `"provenance.aidevops.sh"`
3. Distinctive function names unique to this project
4. Natural language markers: specific comment phrases from code-standards.md
5. Error message strings that are unique to this codebase
6. Structural patterns: `_check_origin`, `TRUSTED_FINGERPRINT`, etc.

### Canary Dashboard

Read from the Cloudflare KV namespace (`3a58e39603844456937cfc44c89993d6`):
- List all `latest:*` keys — each represents a unique non-canonical origin
- Decode the stored JSON: `{h: SHA256(remote_url), v: version, ts: ISO, cf: {co, ci}}`
- Present as a table: hash prefix, version, last seen, country, city
- Alert on new origins (not in `known-forks.json`)

### Implementation Steps

1. Create private repo: `gh repo create marcusquinn/aidevops-provenance --private`
2. Move `MANIFEST.md` from agent workspace to the repo
3. Copy Worker source as a backup
4. Write `search-github.sh` using `gh api search/code` endpoint
5. Write `read-canary-kv.sh` using Cloudflare API (needs API token with KV read)
6. Write `generate-report.sh` combining both
7. Create GitHub Actions workflow for weekly detection
8. Create local launchd plist `sh.aidevops.provenance-scan` for on-demand runs
9. Store a Cloudflare API token as a GitHub Actions secret for KV access

### Credentials Needed

- **GitHub token**: Already available via `gh auth` — `gh api search/code` works with default scopes
- **Cloudflare API token**: Needs a scoped token with KV read access to namespace `3a58e39603844456937cfc44c89993d6`. Create via Cloudflare dashboard → API Tokens → Custom Token → Account:Workers KV Storage:Read

## Acceptance Criteria

1. Private repo exists at `marcusquinn/aidevops-provenance`
2. `search-github.sh` finds the aidevops repo itself when run (baseline validation)
3. `read-canary-kv.sh` successfully reads from the KV namespace
4. `generate-report.sh` produces a human-readable summary
5. GitHub Actions workflow runs on schedule (weekly) and stores reports
6. `known-forks.json` excludes legitimate forks from alerts
7. MANIFEST.md is comprehensive and current

## Context

- Cloudflare Worker: `provenance-ping` at `provenance.aidevops.sh`
- KV namespace ID: `3a58e39603844456937cfc44c89993d6`
- Cloudflare account ID: `18b144721614d93010eb869196ab0c1c`
- Worker source: `~/.aidevops/.agent-workspace/work/provenance-worker/`
- Current MANIFEST: `~/.aidevops/.agent-workspace/work/provenance-worker/MANIFEST.md`
- GitHub Code Search API: `gh api search/code?q=...` — rate-limited to 10 requests/minute for authenticated users, 30 results per page
- The detection repo should NOT be referenced from the public aidevops repo in any way
