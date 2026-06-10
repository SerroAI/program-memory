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

## What goes in your program-memory repo

```
program-memory/
  CLAUDE.md                          ← how Claude answers program questions
  INGEST_PROMPT.md                   ← one-shot ingestion instructions the cron passes to claude
  programs.md                        ← active programs and owners
  programs_to_sources_mapping.yaml   ← which sources belong to each program
  digests/                           ← written by claude on each cron run, one file per run
```

`CLAUDE.md` and `programs_to_sources_mapping.yaml` are the same as Family B — see [B2](../family_b/instructions.md#b2--create-a-shared-program-memory-repo) and [B3](../family_b/instructions.md#b3--build-the-mapping-file).

`INGEST_PROMPT.md` contains the instructions Claude follows on each cron-triggered run. The cron script passes this file as the prompt when invoking `claude` non-interactively. Copy this into your repo and edit the header to match your org:

---

**`INGEST_PROMPT.md`** — copy into your `program-memory` repo:

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

## Step 3 — Pull signals for each program

For each program in the mapping, pull only what's new since `last_run`:

**GitHub** — search declared repos for:
- Commits merged to main
- PRs opened, merged, or closed
- Issues opened or closed
Include: title, author, PR/issue number, and a one-line summary.

**Slack** — search declared channels for:
- Threads with decisions, blockers, or scope changes
Do not surface every message — only notable discussions.

**Drive** — search declared folders for:
- Documents created or modified since last_run
Note: doc title and what section changed if visible.

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

The cron script invokes Claude with this file:

```bash
#!/bin/bash
# ingest.sh — run by cron on a schedule
cd ~/program-memory
git pull

claude -p "$(cat INGEST_PROMPT.md)"

# claude writes the digest, commits, and pushes as instructed
```

Cron entry for hourly runs:

```
0 * * * * /bin/bash ~/program-memory/ingest.sh >> ~/program-memory/logs/cron.log 2>&1
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
