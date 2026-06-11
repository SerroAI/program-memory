# Level 4 Upgrade — LaserData

> **Scope note:** This guide is a pointer, not a validated implementation. Level 4 is outside the core scope of this repo. The instructions below describe how LaserData fits into the pipeline architecturally, but this has not been tested against a real org. Treat it as a starting point for your own evaluation, not a step-by-step you can follow blindly.
>
> If you need production-grade semantic search and entity resolution without building it yourself, [Serro](https://serro.ai) covers this out of the box.

**Prerequisite:** A running Level 3 loop (Option C-4, C-2, or C-3) with a stable `digests/` folder. Add this after your loop has been running long enough that you're hitting the ceiling of flat keyword queries.

---

## What LaserData does

[LaserData](https://laserdata.ai) is a managed data indexing platform for AI pipelines. In this pipeline it plays the same role as CocoIndex — reads your digest files, chunks and embeds them, and makes the result queryable — but as a hosted service rather than a library you run yourself.

The tradeoff vs. CocoIndex: less to operate and configure, less control over chunking strategy and embedding model, and data leaves your environment (relevant if you have compliance requirements).

The queries it enables are the same:

- *"How has the auth approach evolved over the last quarter?"*
- *"Which programs have discussed rate limiting?"*
- *"What decisions are conceptually similar to the one we made about the API gateway?"*

## Pipeline

```
Loop writes digests/YYYY-MM-DD-HH.md
                ↓
LaserData connector watches digests/ (or receives push via webhook)
                ↓
  Chunks by program section
  Embeds and indexes
                ↓
LaserData-hosted vector index
                ↓
Claude queries via LaserData API or MCP connector
→ semantic search over all program memory
```

---

## Setup

**Step 1 — Create a LaserData account**

Sign up at [laserdata.ai](https://laserdata.ai) and create a new index for `program_memory`.

**Step 2 — Connect your digests/ folder**

LaserData supports two connection modes:

*A — File sync (simpler):* Point LaserData at your `digests/` directory. It polls for new files on a configurable interval (5–15 min). No code required — configure in the LaserData dashboard.

*B — Push on write (tighter):* After each loop iteration, push the new digest file directly:

```bash
# Add to ingest.sh or LOOP.md Step 7
curl -X POST https://api.laserdata.ai/v1/documents \
  -H "Authorization: Bearer $LASERDATA_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"index\": \"program_memory\",
    \"source\": \"$(cat digests/$(ls digests/ | tail -1))\",
    \"metadata\": {
      \"filename\": \"$(ls digests/ | tail -1)\",
      \"ingested_at\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
    }
  }"
```

Store `LASERDATA_API_KEY` as an environment variable, not in the repo.

**Step 3 — Configure chunking**

In the LaserData dashboard, set the chunking strategy for your index:

- **Separator**: `## ` (splits by program section heading)
- **Chunk size**: 1000 tokens
- **Overlap**: 100 tokens

This ensures each chunk corresponds to one program's digest block, keeping semantic queries program-scoped.

**Step 4 — Verify the index**

Run a test query from the LaserData dashboard:

```
"what has changed in auth recently"
```

You should see results from digest sections tagged to the auth program. If results are mixing programs, check the chunking separator — it should split on `## [program-name]` headings, not on arbitrary token boundaries.

---

## Querying the index

Update `CLAUDE.md` so Claude queries LaserData for historical or semantic questions:

```markdown
## Querying program memory

For current state: read the most recent file in digests/.

For semantic or historical queries ("how has X evolved", "which programs mention Y",
"decisions similar to Z"): query the LaserData API.

Endpoint: https://api.laserdata.ai/v1/query
Index: program_memory
Auth: LASERDATA_API_KEY environment variable

When querying:
- Pass the user's question as the query string
- Filter by program name in metadata if the question is program-specific
- Request top 10 results, then synthesize
```

If you have a LaserData MCP connector, configure it via `claude mcp add laserdata` and Claude will query it directly without needing the API details in CLAUDE.md.

---

## Keeping the index current

LaserData's file sync polls on an interval — no action required once connected. If you use the push approach (Step 2B), the loop updates the index on each iteration.

For the push approach, add to `LOOP.md` as Step 7:

```markdown
## Step 7 — Push digest to LaserData

After committing and pushing the digest, send the new file to the LaserData index:

LATEST=$(ls digests/ | tail -1)
curl -X POST https://api.laserdata.ai/v1/documents \
  -H "Authorization: Bearer $LASERDATA_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"index\": \"program_memory\", \"source\": \"$(cat digests/$LATEST)\", \"metadata\": {\"filename\": \"$LATEST\"}}"
```

---

## What this unlocks vs. stays the same

| | Before (Level 3) | After (Level 3 + LaserData) |
|---|---|---|
| "What shipped last week in auth?" | ✅ Read latest digest | ✅ Same |
| "How has the auth approach evolved?" | ❌ | ✅ Semantic search across all digest history |
| "Which programs have discussed rate limiting?" | ⚠️ Grep | ✅ One semantic query |
| Entity resolution | ❌ | ❌ Still needs graph layer |
| Data stays on-prem | ✅ | ❌ Digest content sent to LaserData |
| Setup cost | Low | Low-medium — dashboard config, no local infra |

---

## Comparison with CocoIndex

See [`cocoindex_upgrade.md`](cocoindex_upgrade.md) for the alternative. The short version:

- **LaserData**: managed service, minimal config, data leaves your environment
- **CocoIndex**: open-source, runs locally, more control over chunking and embedding model

Use LaserData if you want the simplest path to semantic search and don't have on-prem data requirements. Use CocoIndex if you need full control or data residency.
