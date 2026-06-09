# C2 — Shared Git Repo + Scheduled Claude Agent

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

## Why the git layer matters
- Every memory update is a commit — full history of how org memory evolved over time
- `git diff` shows exactly what was learned between any two runs
- Any agent on any machine reads the same state via `git pull`
- Human-readable, auditable, no database required

## Pros
- Zero new infrastructure — just a cron and a git repo
- Fully Claude Code native
- Git history gives you versioned memory for free
- Easiest migration path to C3 (storage layer stays identical)

## Cons
- Polling — up to 60-minute lag (tunable: run every 15 min if needed)
- Still uses `programs_to_sources_mapping.yaml` to know what to poll — technically a hybrid of Family B and B
- Slack/Drive polling limited by API search constraints (Slack free tier: 90-day history)
- Frequent small commits — noisy git history

## Verdict
Most attainable right now. Accepts polling latency as a known tradeoff. **Start here.**

## Implementation
See [`../../family_c/c2_git_cron/instructions.md`](../../family_c/c2_git_cron/instructions.md).
