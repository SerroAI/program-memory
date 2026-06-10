# Family C - Auto-Ingestion Overview

> Signals are discovered and ingested automatically. No human maintains a source mapping. Coverage is determined by the system.

---

## Four Options at a Glance

| | Option C-1 - Webhook Server | Option C-2 - Git + Cron | Option C-3 - GitHub Actions | Option C-4 - Loop |
|---|---|---|---|---|
| **Latency** | Seconds | Up to 60 min | 1–2 min | Self-paced (15 min–2 hr) |
| **Infrastructure you operate** | Always-on server | Nothing (just cron) | Cloudflare Worker (~10 lines) | Persistent Claude Code process |
| **GitHub signals** | Webhook → server | Polling via MCP | Native trigger on push/PR | Polling via MCP |
| **Slack signals** | Slack Events API → server | Polling via MCP | Slack Events → Worker → GH dispatch | Polling via MCP |
| **Drive/Meet signals** | Drive push notifications → server | Polling via MCP | Drive push → Worker → GH dispatch | Polling via MCP |
| **Memory storage** | Local files or git | Git repo (.md files) | Git repo (.md files) | Git repo (.md files) |
| **Agents read memory via** | Local file read | `git pull` | `git pull` | `git pull` |
| **Claude Code native?** | Partially | Yes | Partially | Fully |
| **Complexity** | High | Low | Medium | Lowest |

## Which to Pick

```
Start with Option C-4 (simplest) or C-2 (headless)
→ migrate to Option C-3 when lag matters
→ Option C-1 only if you need sub-10s latency
```

**Option C-4** is the most Claude-native option and has the lowest setup cost — one `/loop` command, no cron, no bash scripts. The constraint: it requires a persistent Claude Code process. Use it if you're already running Claude Code or want the simplest possible path.

**Option C-2** is the right choice when you need fully headless operation — a cron fires Claude as a subprocess, no active session required. Same git storage layer as C-4, predictable run times.

**Option C-3** is the right upgrade from C-2 or C-4 when hourly lag becomes a real problem. The only new piece you write is a ~10-line Cloudflare Worker.

**Option C-1** is only justified if you need seconds-level latency and are willing to operate a server 24/7.

## Detailed Docs

- [Option C-1 - Webhook Server](c1_webhook_server.md)
- [Option C-2 - Git + Cron](c2_git_cron.md)
- [Option C-3 - GitHub Actions + Serverless Forwarder](c3_github_actions.md)
- [Option C-4 - Claude Code Loop](c4_loop.md)
