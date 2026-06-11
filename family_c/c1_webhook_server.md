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
                              git commit + push to shared serro-diy repo
                                                  ↓
                              Any agent: git pull → reads updated memory
```

## What goes in your serro-diy repo

```
serro-diy/
  CLAUDE.md               ← how Claude answers program questions
  INGEST_PROMPT.md        ← ingestion instructions the server passes to claude per event
  program_mappings.yaml   ← owner, charter, people, and sources per program
  digests/                ← written by claude on each webhook-triggered run
```

`CLAUDE.md` and `program_mappings.yaml` are the same as Family B — see [B2](../family_b/instructions.md#b2--create-a-shared-serro-diy-repo) and [B3](../family_b/instructions.md#b3--build-the-mapping-file).

**MCP configuration:** Configure GitHub, Slack, and Drive connectors in Claude Code settings → Connectors (or `claude mcp add`) before running the server. Claude Code persists these in your user config; the `claude -p` subprocess inherits them automatically.

`INGEST_PROMPT.md` is the ingestion instruction file the server reads and passes to `claude` on each webhook event. It's the same format as [Option C-2's INGEST_PROMPT.md](c2_git_cron.md#what-goes-in-your-serro-diy-repo) — which includes the MCP connectivity check — copy that template, then add an event context section at the end:

```markdown
## Event context

A webhook event just fired. The event payload is appended below. Use it to determine which
program this event belongs to, prioritize pulling full context for that program from MCP,
then run the full ingestion pass for any other programs with pending signals.
```

The server invokes claude per event:

```javascript
// server.js (Express)
const { execSync } = require('child_process')
const fs = require('fs')

app.post('/webhook/:source', verifySignature, async (req, res) => {
  res.sendStatus(200) // ack immediately — don't block webhook sender

  const source = req.params.source        // "github" | "slack" | "drive"
  const payload = JSON.stringify(req.body)
  const basePrompt = fs.readFileSync('/path/to/serro-diy/INGEST_PROMPT.md', 'utf8')
  const fullPrompt = `${basePrompt}\n\nEvent source: ${source}\nPayload: ${payload}`

  execSync(`claude -p "${fullPrompt.replace(/"/g, '\\"')}"`, {
    cwd: '/path/to/serro-diy',
    env: { ...process.env }
  })
})
```

The server handles three webhook sources: GitHub (`/webhook/github`), Slack Events API (`/webhook/slack`), Google Drive push notifications (`/webhook/drive`). Drive push notification channels expire every 24h and need renewal — add a daily job to re-register them.

---

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
The setup steps above are the implementation. For shared prerequisites (MCP setup, repo creation, mapping file), see [`../instructions.md`](instructions.md).
