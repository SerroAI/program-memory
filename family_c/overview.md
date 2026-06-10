# Family C - Auto-Ingestion Overview

> Signals are discovered and ingested automatically. No human maintains a source mapping. Coverage is determined by the system.

---

## Three Options at a Glance

| | Option C-1 - Webhook Server | Option C-2 - Git + Cron | Option C-3 - GitHub Actions |
|---|---|---|---|
| **Latency** | Seconds | Up to 60 min | 1–2 min |
| **Infrastructure you operate** | Always-on server | Nothing (just cron) | Cloudflare Worker (~10 lines) |
| **GitHub signals** | Webhook → server | Polling via MCP | Native trigger on push/PR |
| **Slack signals** | Slack Events API → server | Polling via MCP | Slack Events → Worker → GH dispatch |
| **Drive/Meet signals** | Drive push notifications → server | Polling via MCP | Drive push → Worker → GH dispatch |
| **Memory storage** | Local files or git | Git repo (.md files) | Git repo (.md files) |
| **Agents read memory via** | Local file read | `git pull` | `git pull` |
| **Claude Code native?** | Partially | Yes | Partially |
| **Complexity** | High | Low | Medium |

## Which to Pick

```
Start with Option C-2 → migrate to Option C-3 when lag matters → Option C-1 only if you need sub-10s latency
```

**Option C-2** works today with zero new infrastructure. The git-as-memory-store pattern carries forward cleanly if you later switch to Option C-3 — the storage layer doesn't change, only the trigger mechanism does.

**Option C-3** is the right upgrade path when hourly polling becomes a real problem. The only new piece you write is a ~10-line Cloudflare Worker.

**Option C-1** is only justified if you need seconds-level latency and are willing to operate a server 24/7.

## Detailed Docs

- [Option C-1 - Webhook Server](c1_webhook_server.md)
- [Option C-2 - Git + Cron](c2_git_cron.md)
- [Option C-3 - GitHub Actions + Serverless Forwarder](c3_github_actions.md)
