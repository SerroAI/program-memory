# Option C-3 - GitHub Actions + Stateless Serverless Forwarder

**Latency:** 1–2 min | **Complexity:** Medium | **Infrastructure:** ~10-line Cloudflare Worker

## What it is
GitHub Actions serves as the compute layer - no server to operate. GitHub events trigger Actions natively. Slack and Drive events are forwarded to GitHub via a tiny stateless Cloudflare Worker that POSTs to GitHub's `repository_dispatch` API. The Action runs `claude ingest`, writes to .md files, and commits back to the repo.

## Architecture
```
── GitHub events (PR, push, issue) ──────────────────────────────────┐
                                                                      │ native trigger
── Slack Events API ──▶ Cloudflare Worker ──▶ repository_dispatch ───┤
                          (~10 lines, free)     POST to GitHub API    │
── Drive push notif ──▶ same Worker ────────────────────────────────┘
                                                                      ↓
                                                    GitHub Actions runner (cloud VM)
                                                                      ↓
                                                    claude ingest "${{ github.event.payload }}"
                                                                      ↓
                                                    Claude uses MCP to fetch full context
                                                                      ↓
                                                    writes to programs/<name>/signals/YYYY-MM.md
                                                                      ↓
                                                    git commit + push (back to same repo)
                                                                      ↓
                                                    agents: git pull → read updated memory
```

## Key distinction from Option C-1
GitHub Actions runs in GitHub's cloud - it never touches your local machine. The only thing you write and own is the Cloudflare Worker. GitHub manages the compute.

## Pros
- Near-real-time for GitHub events (triggers instantly on push/PR)
- Near-real-time for Slack/Drive with Worker in place (~1–2 min roundtrip)
- No always-on server to operate
- Free within GitHub Actions free tier (2,000 min/month)
- Cron fallback in the workflow catches anything missed by event triggers
- Cloudflare Worker free tier: 100k requests/day

## Cons
- More moving parts than Option C-2 (Worker + Actions workflow + secrets)
- Claude Code CLI install adds ~30s to each Actions run
- MCP servers need configuring in the runner environment
- ~30s GitHub Actions cold start - not sub-second

## Verdict
Best balance of latency and operational simplicity for teams already on GitHub. Upgrade from Option C-2 when hourly polling becomes a problem.

## Implementation
See [`../../family_c/c3_github_actions/instructions.md`](../../family_c/c3_github_actions/instructions.md).
