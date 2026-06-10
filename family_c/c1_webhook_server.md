# Option C-1 - Webhook Server

**Latency:** Seconds | **Complexity:** High | **Infrastructure:** Always-on server

## What it is
An always-on HTTP server you run that receives real-time events from GitHub, Slack, and Google Drive webhooks. On each event it triggers `claude ingest <payload>` via CLI, which uses MCP tools to fetch full context and writes to the memory store.

## Architecture
```
GitHub webhook ─────────────────────────────────┐
Slack Events API ──────────────────────────────▶ Express/FastAPI server (always-on)
Google Drive push notifications ─────────────────┘
                                                  ↓
                                        claude ingest "<payload>"
                                                  ↓
                              Claude uses MCP to fetch full context
                                                  ↓
                              Writes entry to programs/<name>/signals/YYYY-MM.md
                                                  ↓
                              git commit + push to shared org-memory repo
                                                  ↓
                              Any agent: git pull → reads updated memory
```

## Pros
- Real-time (seconds latency)
- Full architectural parity with how Serro likely works
- Handles all three sources uniformly

## Cons
- You operate a server 24/7 - uptime, restarts, monitoring are your problem
- Webhook signature validation required for security (boilerplate but necessary)
- Drive push notification channels expire every 24h and need renewal

## Verdict
Highest fidelity. Most operational overhead. Right choice only if latency matters and you're willing to run infrastructure.

## Implementation
See [`../../family_a/c1_webhook_server/instructions.md`](../../family_a/c1_webhook_server/instructions.md).
