# Option C-2 - Shared Git Repo + Scheduled Claude Agent

**Latency:** Up to 60 min | **Complexity:** Low | **Infrastructure:** Nothing (just cron)

## What it is
A git repo holds all program memory as .md files. A scheduled Claude agent wakes up on an interval, polls the MCP integrations for recent signals, classifies them into programs, writes new entries to the .md files, then commits and pushes. Any agent anywhere does `git pull` to read the latest state.

## Architecture
```
Cron fires (hourly or daily)
        ↓
Claude wakes up, reads programs_to_sources_mapping.yaml
        ↓
For each program:
  - GitHub MCP: "list PRs merged in last N hours for these repos"
  - Slack MCP: "search messages in these channels since last run"
  - Drive MCP: "list files modified in these folders since last run"
        ↓
Classifies each signal into a program
Writes structured entries to programs/<name>/signals/YYYY-MM.md
Updates .last_run timestamp
        ↓
git commit + git push
        ↓
Any agent anywhere: git pull → reads updated memory
```

## What goes in your serro-diy repo

```
serro-diy/
  CLAUDE.md                          ← how Claude answers program questions
  INGEST_PROMPT.md                   ← one-shot ingestion instructions the cron passes to claude
  programs.md                        ← active programs and owners
  programs_to_sources_mapping.yaml   ← which sources belong to each program
  digests/                           ← written by claude on each cron run, one file per run
```

`CLAUDE.md` and `programs_to_sources_mapping.yaml` are the same as Family B — see [B2](../family_b/instructions.md#b2--create-a-shared-serro-diy-repo) and [B3](../family_b/instructions.md#b3--build-the-mapping-file).

`INGEST_PROMPT.md` contains the instructions Claude follows on each cron-triggered run. The cron script passes this file as the prompt when invoking `claude` non-interactively. Copy this into your repo and edit the header to match your org:

---

**`INGEST_PROMPT.md`** — copy into your `serro-diy` repo:

```markdown
# Ingestion Instructions

You are the program memory agent for this org. Execute these steps once and exit.

---

## Step 1 — Read the source mapping

Read `programs_to_sources_mapping.yaml` to get the list of active programs and their
declared sources (GitHub repos, Slack channels, Drive folders).

## Step 2 — Find the last run timestamp

Read the most recent file in `digests/` and note its `last_run` timestamp.
If no digest exists yet, treat `last_run` as 24 hours ago.

## Step 2b — Check MCP connectivity

Before pulling any signals, verify that each MCP server is reachable. For any server
that is unavailable, skip all sources it handles for every program and write
`[MCP unavailable — skipped]` in those sections of the digest. Do not attempt to call
a disconnected server.

## Step 3 — Pull signals for each program

For each program in the mapping, pull only what's new since `last_run`. Pass the
timestamp as a filter directly to each MCP tool call — do not pull full history and
filter after the fact.

**GitHub** — for each declared repo:
- Commits merged to main: pass `since` = `last_run` (ISO 8601, e.g. `2024-01-15T14:30:00Z`)
- PRs opened, merged, or closed: filter by `updated_at >= last_run` (ISO 8601)
- Issues opened or closed: pass `since` = `last_run` (ISO 8601)
Include: title, author, PR/issue number, one-line summary.

**Slack** — for each declared channel:
- Pass `oldest` = `last_run` converted to a Unix timestamp (integer seconds since epoch)
  Example: `2024-01-15T14:30:00Z` → `1705329000`
- Surface threads with decisions, blockers, or scope changes only — not every message.

**Drive** — for each declared folder:
- Query `modifiedTime > '[last_run]'` (RFC 3339 format; ISO 8601 is compatible)
  Example: `modifiedTime > '2024-01-15T14:30:00Z'`
- Note: doc title and what changed, if visible.

If a source is unreachable, write `[source unreachable — skipped]` under that section
rather than leaving it blank. Do not retry — move on and note it.

## Step 4 — Write the digest

Create a new file at `digests/YYYY-MM-DD-HH.md` with this structure for every program
in the mapping (include programs with no activity — do not skip them):

---
last_run: [ISO 8601 timestamp]

## [program-name]
**GitHub**: [what shipped or changed — titles, numbers, authors]
**Slack**: [key decisions, blockers, or threads worth noting]
**Drive**: [docs updated and what changed]
**Status**: [one sentence on where this program stands right now]
**Watch**: [open blockers or unresolved decisions — leave blank if none]

Write "No activity." under any section with nothing to report.

## Step 5 — Commit and push

git add digests/
git commit -m "digest: [YYYY-MM-DD HH:MM]"
git push
```

---

**MCP configuration**

Configure your GitHub, Slack, and Drive connectors in Claude Code before running the
cron job. Open Claude Code settings → Connectors (or run `claude mcp add`) and connect
each account. Claude Code persists these in your user config and uses them automatically
when running headlessly — no separate secrets or config files needed.

The cron script invokes Claude with this file:

```bash
#!/bin/bash
# ingest.sh — run by cron on a schedule
cd ~/serro-diy
git pull

claude -p "$(cat INGEST_PROMPT.md)"

# claude writes the digest, commits, and pushes as instructed
```

Cron entry for hourly runs:

```
0 * * * * /bin/bash ~/serro-diy/ingest.sh >> ~/serro-diy/logs/cron.log 2>&1
```

For 15-minute intervals: `*/15 * * * *`.

---

## Why the git layer matters
- Every memory update is a commit - full history of how org memory evolved over time
- `git diff` shows exactly what was learned between any two runs
- Any agent on any machine reads the same state via `git pull`
- Human-readable, auditable, no database required

## Pros
- Zero new infrastructure - just a cron and a git repo
- Fully Claude Code native
- Git history gives you versioned memory for free
- Easiest migration path to Option C-3 (storage layer stays identical)

## Cons
- Polling - up to 60-minute lag (tunable: run every 15 min if needed)
- Still uses `programs_to_sources_mapping.yaml` to know what to poll - technically a hybrid of Family B and B
- Slack/Drive polling limited by API search constraints (Slack free tier: 90-day history)
- Frequent small commits - noisy git history

## Verdict
Most attainable right now. Accepts polling latency as a known tradeoff. **Start here.**

## Implementation
See [`../../family_c/c2_git_cron/instructions.md`](../../family_c/c2_git_cron/instructions.md).
