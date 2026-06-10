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

## What goes in your program-memory repo

```
program-memory/
  CLAUDE.md                          ← how Claude answers program questions
  INGEST_PROMPT.md                   ← ingestion instructions the GitHub Action passes to claude
  programs.md                        ← active programs and owners
  programs_to_sources_mapping.yaml   ← which sources belong to each program
  digests/                           ← written by claude on each triggered run
  .github/
    workflows/
      ingest.yml                     ← the Action that runs claude on each event
```

`CLAUDE.md` and `programs_to_sources_mapping.yaml` are the same as Family B — see [B2](../family_b/instructions.md#b2--create-a-shared-program-memory-repo) and [B3](../family_b/instructions.md#b3--build-the-mapping-file).

`INGEST_PROMPT.md` is the ingestion instruction file the GitHub Action reads and passes to `claude`. It's the same format as [Option C-2's INGEST_PROMPT.md](c2_git_cron.md#what-goes-in-your-program-memory-repo) — copy that template, then add an event context instruction at the end:

```markdown
## Event context (appended by the Action at runtime)

An event just fired. The event payload is appended below. Use it to prioritize
which program to pull signals for first — but still run the full ingestion pass
for all programs.
```

The GitHub Action appends the event payload when invoking claude:

```yaml
# .github/workflows/ingest.yml
name: Ingest program signals

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, closed, merged]
  repository_dispatch:     # receives forwarded Slack + Drive events from the Worker
    types: [slack_event, drive_event]
  schedule:
    - cron: '0 * * * *'   # hourly fallback for anything missed by event triggers

jobs:
  ingest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code

      - name: Install MCP servers
        run: |
          npm install -g @modelcontextprotocol/server-github
          npm install -g @modelcontextprotocol/server-slack
          npm install -g @modelcontextprotocol/server-gdrive

      - name: Run ingestion
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
        run: |
          PROMPT="$(cat INGEST_PROMPT.md)"
          PAYLOAD='${{ toJson(github.event.client_payload) }}'
          claude -p "${PROMPT}

          Event payload: ${PAYLOAD}"

      - name: Push digest
        run: |
          git config user.email "actions@github.com"
          git config user.name "Ingest Agent"
          git push
```

The Cloudflare Worker (~10 lines) forwards Slack and Drive events to `repository_dispatch`:

```javascript
// worker.js
export default {
  async fetch(req, env) {
    const payload = await req.json()
    const source = new URL(req.url).pathname.slice(1) // "slack" or "drive"

    await fetch(`https://api.github.com/repos/${env.REPO}/dispatches`, {
      method: 'POST',
      headers: {
        Authorization: `token ${env.GH_TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ event_type: `${source}_event`, client_payload: payload }),
    })

    return new Response('ok')
  }
}
```

---

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
