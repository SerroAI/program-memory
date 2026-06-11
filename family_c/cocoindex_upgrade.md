# Level 4 Upgrade — CocoIndex

**Prerequisite:** A running Level 3 loop (Option C-4, C-2, or C-3) with a stable `digests/` folder. Add this after your loop has been running long enough that you're hitting the ceiling of flat keyword queries.

---

## What CocoIndex does

[CocoIndex](https://github.com/cocoindex-io/cocoindex) is an open-source incremental data transformation framework. In this pipeline it plays one role: reads the digest files your loop writes, splits them by program section, generates embeddings, and pushes the result into a vector store.

Without CocoIndex, Claude reads raw markdown files and can only keyword-match. With CocoIndex, Claude queries a vector index and can answer:

- *"How has the auth approach evolved over the last quarter?"*
- *"Which programs have discussed rate limiting?"*
- *"What decisions are conceptually similar to the one we made about the API gateway?"*

## Pipeline

```
Loop writes digests/YYYY-MM-DD-HH.md
                ↓
CocoIndex watches digests/ (incremental — only processes new/changed files)
                ↓
  Split each digest by ## [program-name] section
  Generate embedding per section
  Store: content + embedding + program + timestamp metadata
                ↓
Vector store (Chroma / pgvector / Pinecone)
                ↓
Claude queries vector store via MCP or direct client
→ semantic search over all program memory
```

CocoIndex is incremental: it tracks which digest files it has already processed. Each loop run adds one new file to `digests/` — CocoIndex picks it up and indexes only that file, not the full history.

---

## Setup

**Step 1 — Install CocoIndex**

```bash
pip install cocoindex
```

Requires Python 3.10+.

**Step 2 — Choose a vector store**

| Store | When to use |
|---|---|
| [Chroma](https://www.trychroma.com) | Local dev, no ops — runs in-process or as a server |
| [pgvector](https://github.com/pgvector/pgvector) | Already running Postgres — zero new infra |
| [Pinecone](https://www.pinecone.io) | Managed, no ops, higher scale |

For most teams starting out: Chroma (zero setup) or pgvector (if Postgres is already running).

**Step 3 — Write the CocoIndex flow**

Create `cocoindex_flow.py` in your `serro-diy` repo:

```python
import cocoindex

@cocoindex.flow_def(name="program_memory")
def program_memory_flow(flow_builder: cocoindex.FlowBuilder, data_scope: cocoindex.DataScope):
    # Source: watch the digests/ directory for new or changed files
    digests = flow_builder.add_source(
        cocoindex.sources.LocalFile(
            path="digests/",
            included_patterns=["*.md"],
        )
    )

    # Split each digest file into per-program sections
    # Each ## heading in a digest file is one program's block
    program_sections = digests.transform(
        cocoindex.functions.SplitRecursively(
            language="markdown",
            chunk_size=1000,
            chunk_overlap=100,
        )
    )

    # Embed each section
    embeddings = program_sections.transform(
        cocoindex.functions.SentenceTransformerEmbed(
            model="sentence-transformers/all-MiniLM-L6-v2"
        )
    )

    # Export to vector store
    # Swap target for pgvector or Pinecone as needed
    embeddings.export(
        "program_sections",
        cocoindex.targets.Chroma(
            collection_name="program_memory",
            path=".chroma",         # local disk; point to a server URL for shared access
        ),
        primary_key_fields=["filename", "chunk_index"],
        vector_index_params=cocoindex.VectorIndexParams(
            metric=cocoindex.VectorSimilarityMetric.COSINE,
        ),
    )
```

**Step 4 — Run the initial index**

```bash
cd ~/serro-diy
python cocoindex_flow.py cocoindex update
```

This indexes all existing digest files. Subsequent runs are incremental — only new/changed files are processed.

**Step 5 — Keep the index current**

Two options:

*A — Run after each loop iteration.* Add to your `LOOP.md` or `INGEST_PROMPT.md` as Step 7:

```markdown
## Step 7 — Update the index

After pushing the digest, run:
python cocoindex_flow.py cocoindex update
```

*B — Run on a separate cron.* A lightweight process that polls `digests/` every few minutes:

```
*/5 * * * * cd ~/serro-diy && python cocoindex_flow.py cocoindex update >> logs/cocoindex.log 2>&1
```

Option A keeps the index tighter but adds latency to each loop iteration. Option B decouples them — prefer this if your loop already self-paces.

---

## Querying the index

Once indexed, update `CLAUDE.md` so Claude queries the vector store for historical or semantic questions:

```markdown
## Querying program memory

For current state: read the most recent file in digests/.

For semantic or historical queries ("how has X evolved", "which programs mention Y",
"decisions similar to Z"): query the Chroma collection at .chroma/
using the program_memory collection.

When querying semantically, include:
- The query embedding
- Metadata filter: program name if the question is program-specific
- Top-k: 10 results, then synthesize across them
```

If teammates access the index remotely, point Chroma at a shared server URL instead of `.chroma/`.

---

## What this unlocks vs. stays the same

| | Before (Level 3) | After (Level 3 + CocoIndex) |
|---|---|---|
| "What shipped last week in auth?" | ✅ Read latest digest | ✅ Same — keyword is fine here |
| "How has the auth approach evolved?" | ❌ Keyword can't answer this | ✅ Semantic search across all digest history |
| "Which programs have discussed rate limiting?" | ⚠️ Keyword grep across all digests | ✅ One semantic query |
| Entity resolution (same person across tools) | ❌ | ❌ Still not solved — needs graph layer (FalkorDB) |
| Temporal point-in-time queries | ❌ | ⚠️ Partial — each digest is timestamped, but no snapshot diffing |
| Setup cost | Low | Medium — Python + vector store |

Entity resolution and temporal snapshots are graph DB territory. CocoIndex gets you semantic search; add FalkorDB on top if you need the rest.

---

## Comparison with LaserData

See [`laserdata_upgrade.md`](laserdata_upgrade.md) for the alternative. The short version:

- **CocoIndex**: open-source, more control, runs locally, requires writing the flow config
- **LaserData**: managed, less config, trades control for simplicity

Use CocoIndex if you want full control over chunking, embedding model, and vector store choice. Use LaserData if you want less to operate.
