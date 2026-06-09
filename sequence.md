# Implementation Sequence

> High-level steps for each family. Iterate here before breaking into individual READMEs.
> Each step will become its own file. Steps with multiple technology options are marked with ⚙️.

---

## Family A — Full Context Pull

For orgs with ≤5 engineers, 1 repo, 1–2 Slack channels, 1–2 programs. No config, no mapping, no ingestion.

| Step | Name | What happens |
|---|---|---|
| A1 | Connect your tools | Install MCP servers for GitHub, Slack, Drive, and meetings |
| A2 | Name your programs | Write a short list of your active programs — no yaml, just names and one-line descriptions |
| A3 | Write your CLAUDE.md | Tells Claude to pull all sources at query time — which repos, which channels, which folders |
| A4 | Run your first query | Test a cross-source question against a real program |
| A5 | Set a growth trigger | Define the signal that tells you to upgrade to Family B (source count, context window failures, latency) |

**Technology options at A1:** GitHub MCP (official), Slack MCP (official), Google Drive MCP (official), Google Meet / transcript source ⚙️

---

## Family B — Manual Source Mapping

For orgs that have outgrown full context pull and are willing to maintain a mapping file. Claude queries only declared sources at query time.

| Step | Name | What happens |
|---|---|---|
| B1 | Connect your tools | Install MCP servers for all sources your programs touch |
| B2 | Name your programs | Define each program with a name, owner, and one-line charter |
| B3 | Build the mapping file | Create `programs_to_sources_mapping.yaml` — for each program, declare its GitHub repos, Slack channels, Drive folders, and meeting series |
| B4 | Write your CLAUDE.md | Tells Claude to read the mapping file and query only the declared sources for the active program |
| B5 | Run the context window test | Fire one real cross-source query per program. Measure tokens, latency, and answer quality against a human ground truth |
| B6 | Set up the mapping health check | A script or scheduled agent that verifies all declared sources still exist and alerts if any are missing ⚙️ |
| B7 | Assign a mapping owner | One named person responsible for updating the mapping when sources change |
| B8 | Define your upgrade trigger | The signal that tells you to move to Family C (context window failures, keyword search gaps, scale) |

**Technology options at B6:** Shell script + cron, GitHub Actions scheduled job, Claude scheduled agent ⚙️

---

## Family C — Auto-Ingestion

For orgs that need semantic search, long history, or high query volume. Signals are ingested into a memory store on a schedule or webhook. Claude queries the store, not the live APIs.

Family C has three sub-options based on latency tolerance. Steps C1–C5 are shared. Steps C6+ diverge.

### Shared steps (all C options)

| Step | Name | What happens |
|---|---|---|
| C1 | Connect your tools | Install MCP servers for all sources — same as Family B |
| C2 | Name your programs | Define programs with names, owners, and charters — same as Family B |
| C3 | Build the mapping file | Create `programs_to_sources_mapping.yaml` — same as Family B |
| C4 | Choose your memory store | Where ingested signals will live and be queried from ⚙️ |
| C5 | Choose your ingestion method | Pick C2 (hourly cron), C3 (GitHub Actions + Worker), or C1 (webhook server) based on latency needs ⚙️ |

**Technology options at C4:** Git repo (markdown files, free, versioned), SQLite, Postgres, vector store (Chroma, Pinecone, pgvector) ⚙️

---

### C2 — Git + Scheduled Cron (recommended start)

Hourly ingestion. Zero new infrastructure beyond a shared git repo. Best starting point.

| Step | Name | What happens |
|---|---|---|
| C2-1 | Create the memory repo | A shared git repo where ingested signals are stored as markdown files, one file per program |
| C2-2 | Write the ingestion agent | A Claude agent that reads the mapping file, queries each source via MCP, and writes a structured summary to the memory repo ⚙️ |
| C2-3 | Schedule the agent | Set up a cron job or scheduled runner that triggers the ingestion agent on a timer ⚙️ |
| C2-4 | Write your CLAUDE.md | Tells Claude to read from the memory repo files instead of querying live sources |
| C2-5 | Run a full ingestion cycle | Trigger manually, verify output files, spot-check 10 signals for accuracy |
| C2-6 | Tune the schedule | Start at hourly. Drop to 15-minute intervals if lag is a real problem |

**Technology options at C2-2:** Claude Code agent, custom script using Anthropic API ⚙️
**Technology options at C2-3:** macOS cron / launchd, GitHub Actions scheduled workflow, Railway, Render cron job ⚙️

---

### C3 — GitHub Actions + Cloudflare Worker

1–2 minute latency on GitHub events. ~10-line Cloudflare Worker forwards Slack and Drive events. No always-on server.

| Step | Name | What happens |
|---|---|---|
| C3-1 | Create the memory repo | Same as C2-1 |
| C3-2 | Write the ingestion agent | Same as C2-2 — triggered by event instead of timer |
| C3-3 | Set up GitHub Actions workflow | On push / PR / release events, trigger the ingestion agent for the affected repo's program ⚙️ |
| C3-4 | Deploy the Cloudflare Worker | A ~10-line Worker that receives Slack and Drive webhooks and triggers the GitHub Actions workflow ⚙️ |
| C3-5 | Register webhooks | Point Slack event subscriptions and Drive push channels at the Worker URL |
| C3-6 | Add a cron fallback | A scheduled GitHub Actions job that runs full ingestion hourly — catches anything the webhooks missed |
| C3-7 | Write your CLAUDE.md | Same as C2-4 |
| C3-8 | Test end-to-end | Push a commit, verify it triggers ingestion within 2 minutes |

**Technology options at C3-3:** GitHub Actions (free tier), self-hosted runner ⚙️
**Technology options at C3-4:** Cloudflare Worker (free tier), Vercel Edge Function, AWS Lambda ⚙️

---

### C1 — Webhook Server (highest fidelity, most ops)

Seconds latency. You operate an always-on server. Highest operational cost.

| Step | Name | What happens |
|---|---|---|
| C1-1 | Create the memory repo | Same as C2-1 |
| C1-2 | Write the ingestion agent | Same as C2-2 |
| C1-3 | Set up the webhook server | An always-on HTTP server that receives events from GitHub, Slack, and Drive ⚙️ |
| C1-4 | Implement webhook validation | Verify signatures from each source before processing |
| C1-5 | Register webhooks | Point GitHub, Slack, and Drive at your server's public URL |
| C1-6 | Set up Drive channel renewal | Drive push channels expire every 24h — automate renewal ⚙️ |
| C1-7 | Set up uptime monitoring | Alert if the server goes down — missed events don't retry ⚙️ |
| C1-8 | Write your CLAUDE.md | Same as C2-4 |
| C1-9 | Test end-to-end | Send a test event from each source, verify ingestion within seconds |

**Technology options at C1-3:** Node.js / Express, Python / FastAPI, Railway, Fly.io, Render ⚙️
**Technology options at C1-6:** GitHub Actions daily cron, server-internal scheduler ⚙️
**Technology options at C1-7:** Better Uptime, UptimeRobot, Datadog ⚙️

---

## Going further — tools worth knowing

These aren't required for any family, but they solve real problems that come up as you scale the ingestion pipeline.

| Tool | What it is | Where it fits |
|---|---|---|
| **[Apache Iggy](https://github.com/iggy-rs/iggy)** | Persistent message streaming platform — think lightweight Kafka, written in Rust | Replace direct webhook-to-agent calls in C1/C3 with a durable message queue. Gives you event replay, backpressure handling, and guaranteed delivery if the ingestion agent is temporarily unavailable |
| **[CocoIndex](https://github.com/cocoindex-io/cocoindex)** | Open-source incremental data transformation framework built for AI indexing pipelines | Replace the custom ingestion agent in C2/C3/C1 with a declarative pipeline. Handles source-to-index transformation, incremental updates, and embedding generation in one place |

If you're finding that your ingestion agent is brittle under load or that re-indexing from scratch is expensive, these are the first two tools to evaluate.

---

## Shared final steps (all families)

After your chosen family is running:

| Step | Name | What happens |
|---|---|---|
| V1 | Validate memory | Run for 30 days on a real program. Spot-check 20 signals for classification accuracy. Target: >80% correct |
| V2 | Measure coverage | Compare ingested signals against ground truth. Identify what's missing and why |
| V3 | Define upgrade trigger | What signal tells you the current family is no longer sufficient |
