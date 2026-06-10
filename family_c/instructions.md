# Family C: Auto-Ingestion — Instructions

> Signals are ingested into a versioned memory store on a schedule or webhook. Claude queries the store — not the live APIs. For orgs needing semantic search, long history, or high query volume.

---

## Choose your option first

Family C has three sub-options. Pick based on how much latency you can tolerate and how much infrastructure you're willing to operate.

| Option | Latency | Infrastructure | Start here if |
|---|---|---|---|
| [Option C-2 — Git + Cron](c2_git_cron.md) | Hourly | Zero beyond a shared git repo | You want the simplest possible start |
| [Option C-3 — GitHub Actions + Cloudflare Worker](c3_github_actions.md) | 1–2 min | ~10-line Cloudflare Worker (free tier) | You need near-real-time without a server |
| [Option C-1 — Webhook Server](c1_webhook_server.md) | Seconds | Always-on server you operate | Seconds matter — live incident response, immediate meeting capture |

**If you're not sure, start with Option C-2.** It's the lowest-complexity path and you can upgrade to Option C-3 or Option C-1 later without changing your memory store or CLAUDE.md.

---

## Shared setup steps

All three options share the same foundation. Complete these before branching to your chosen option.

- [C1 — Connect your tools](#c1--connect-your-tools)
- [C2 — Name your programs and create the shared repo](#c2--name-your-programs-and-create-the-shared-repo)
- [C3 — Build the mapping file](#c3--build-the-mapping-file)
- [C4 — Choose your memory store](#c4--choose-your-memory-store)
- [C5 — Choose your ingestion method](#c5--choose-your-ingestion-method)

---

## C1 — Connect your tools

Install MCP servers for all sources your programs touch. Same as Family A and B.

```bash
claude mcp add github -- npx -y @modelcontextprotocol/server-github
claude mcp add slack -- npx -y @modelcontextprotocol/server-slack
claude mcp add gdrive -- npx -y @modelcontextprotocol/server-gdrive
```

See [Family A — A1](../family_a/instructions.md#a1--connect-your-tools) for environment variable setup, required OAuth scopes, and troubleshooting.

Run `/mcp` to verify all servers are connected.

---

## C2 — Name your programs and create the shared repo

Same as Family B: create a `program-memory` repo, add `programs.md`, `CLAUDE.md`, and `programs_to_sources_mapping.yaml`.

See [Family B — B2](../family_b/instructions.md#b2--create-a-shared-program-memory-repo) for the full setup.

### Connecting teammates to this repo

Every teammate connects once. Two ways:

**Add it as a project in Claude Code**

```
/project:add ~/path/to/program-memory
```

For automated ingestion scripts (Option C-2 cron, Option C-3 Actions), `cd` into the `program-memory` repo before invoking `claude` so the right `CLAUDE.md` is loaded.

---

## C3 — Build the mapping file

Same as Family B. Create `programs_to_sources_mapping.yaml` declaring which sources belong to each program.

See [Family B — B3](../family_b/instructions.md#b3--build-the-mapping-file) for the format and rules.

The ingestion agent reads this file to know what to pull for each program.

---

## C4 — Choose your memory store

The memory store is where ingested signals live. Claude queries this instead of the live APIs.

**Option A — Git repo (recommended start)** ⚙️

A shared git repo where ingested signals are stored as markdown files, one file per program. Free, versioned, no extra infrastructure.

```
memory-store/
├── auth-modernization.md      ← latest ingested summary for this program
├── platform-reliability.md
└── mobile-launch.md
```

Each file is overwritten (or appended) on every ingestion run. Git history gives you a time-series for free.

Best for: Option C-2 and Option C-3. Simple, auditable, no query infrastructure needed.

**Option B — SQLite** ⚙️

A local SQLite file committed to the memory repo or stored on a server. Supports structured queries (by date, by source type, by program).

```sql
CREATE TABLE signals (
  id TEXT PRIMARY KEY,
  program TEXT,
  source_type TEXT,    -- github | slack | drive | meeting
  source_id TEXT,
  content TEXT,
  ingested_at TEXT
);
```

Best for: orgs that want to query across signals (e.g. "all GitHub PRs for this program last week") without re-parsing markdown.

**Option C — Vector store** ⚙️

A vector database that stores signal embeddings alongside content. Enables semantic search — "how has our auth approach evolved" works without keyword matching.

Options:
- [Chroma](https://www.trychroma.com) — open source, runs locally or self-hosted
- [pgvector](https://github.com/pgvector/pgvector) — Postgres extension, if you already run Postgres
- [Pinecone](https://www.pinecone.io) — managed, no ops

Best for: orgs that specifically need semantic search or long-horizon technical reasoning. More setup required.

**Option D — CocoIndex pipeline** ⚙️

[CocoIndex](https://github.com/cocoindex-io/cocoindex) is an open-source incremental data transformation framework built for AI indexing pipelines. It handles source-to-index transformation, incremental updates, and embedding generation declaratively — replacing a custom ingestion agent entirely.

Best for: orgs that want a production-grade pipeline without writing ingestion logic from scratch.

---

## C5 — Choose your ingestion method

Your ingestion method determines how signals get from live sources into your memory store.

| Method | How it triggers | Best for |
|---|---|---|
| [Option C-2 — Scheduled cron](c2_git_cron.md) | Timer (hourly, or every 15 min) | Simplest start, hourly lag acceptable |
| [Option C-3 — GitHub Actions + Worker](c3_github_actions.md) | GitHub push events + Slack/Drive webhooks | 1–2 min lag, no server |
| [Option C-1 — Webhook server](c1_webhook_server.md) | Direct webhooks from all sources | Seconds lag, you operate a server |

**Advanced: Apache Iggy as event bus** ⚙️

For Option C-1 and Option C-3 at scale, consider inserting [Apache Iggy](https://github.com/iggy-rs/iggy) as a durable message queue between your webhook receivers and ingestion agent. Instead of the ingestion agent processing events synchronously, events are written to Iggy and consumed at the agent's pace. Gives you:
- Replay on failure — missed events don't disappear
- Backpressure — ingestion agent isn't overwhelmed by bursts
- Audit log — full event history across all sources

Not needed for most orgs starting out. Evaluate after Option C-1 or Option C-3 is stable and you're seeing dropped events or ingestion failures under load.

---

## Continue to your chosen option

- [Option C-2 — Git + Cron](c2_git_cron.md) ← start here
- [Option C-3 — GitHub Actions + Cloudflare Worker](c3_github_actions.md)
- [Option C-1 — Webhook Server](c1_webhook_server.md)
