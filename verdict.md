# Verdict: Which Level to Use and Why

There are three levels of live program memory. Each is self-contained — you don't need to go further unless you hit the ceiling of the one you're on.

---

## Level 1 — Pull everything

**What it is:** No config. When someone asks a program question, Claude pulls all sources at query time and synthesizes the answer in a single session.

**What you set up:** Connect MCP servers for GitHub, Slack, and Drive. Write a `CLAUDE.md` that tells Claude what to pull and how to answer. That's it.

**The ceiling:** Context window. Once you have more than ~20 sources, or your programs have deep history, the pull starts hitting limits — answers get truncated, latency climbs, and some signals get dropped silently.

**Who it's right for:** Teams of ≤5, one or two active programs, not yet burned by the context ceiling. Start here. It takes 20 minutes to set up.

→ See [Family A instructions](family_a/instructions.md)

---

## Level 2 — Maintain a mapping

**What it is:** You declare programs, sources, and people in `program_mappings.yaml`. Claude queries only the declared sources for the relevant program. You maintain the file.

**What you set up:** `program_mappings.yaml` — one entry per program with owner, charter, contributors, Slack IDs, and source declarations. A `CLAUDE.md` that reads from it. Optional: a daily digest script so Claude pre-processes recent signals rather than pulling live every time.

**What you gain over Level 1:**
- Scoped queries: Claude only touches sources relevant to the asked-about program
- Contributor attribution: `leads`, `contributors`, `slack_ids` are explicit, not inferred
- Action item follow-up: notify targets are declared, not guessed
- Digest caching: Claude reads pre-built context before going live, so queries are faster

**The ceiling:** Human maintenance. The mapping file is a contract — it only stays true if someone keeps it current. When a channel gets renamed, a repo gets archived, or a new person joins a program, someone has to open a PR. If that discipline breaks down, the mapping silently drifts and Claude queries the wrong sources without knowing it.

**Who it's right for:** Teams that have outgrown Level 1 and have a TPM, EM, or program owner willing to own the mapping file. Works best when programs are stable and sources don't change often.

→ See [Family B instructions](family_b/instructions.md)

---

## Level 3 — Automate with loops

**What it is:** A Claude loop automaintains the memory. Instead of you pulling signals or running a nightly script, Claude wakes up on a schedule, reads the mapping, pulls signals from all declared sources since the last run, writes a digest, commits it, and decides when to run again based on what it found.

**What you set up:** Everything from Level 2, plus a `LOOP.md` (or `INGEST_PROMPT.md`) that Claude follows on each iteration. Start the loop with one command:

```
cd ~/serro-diy
claude
/loop Read LOOP.md and follow those instructions on every iteration.
```

That's it. Claude is the ingestion pipeline.

**What you gain over Level 2:**
- Always-current digests: memory updates on a self-paced schedule (30 min–2 hours depending on activity), not whenever someone remembers to run a script
- Self-pacing: more active programs get tighter loops; quiet programs get longer intervals
- No cron or bash: the loop primitive handles scheduling natively
- Persistent context: within a session, the loop remembers what it saw last run — no cold start cost
- Natural migration path: the same `/loop` prompt that runs locally today runs on Anthropic cloud workflows when they ship

**The ceiling:** Flat markdown digests. The loop keeps memory current but queries are keyword-based — you can't ask "how has our auth approach evolved semantically" or "who are the cross-program contributors linking auth and platform." That's Level 4.

**Option C-4 is the right starting point for Level 3.** It has the lowest setup cost of any auto-ingestion option and produces the same output as the more complex options (C-2 cron, C-3 GitHub Actions, C-1 webhook server). Those exist for specific operational requirements — headless execution, near-real-time latency, or CI-native environments. Start with C-4 unless you have a specific reason not to.

→ See [Option C-4 instructions](family_c/c4_loop.md)

---

## Level 4 — Graph index

**What it is:** The loop's digest output is piped into a semantic and graph index. Instead of querying flat markdown files, you query a structured index that understands relationships, embeddings, and time.

**What you set up:** A Level 3 loop (already running), plus one or more of:
- **Vector DB** (Chroma, pgvector, Pinecone) + CocoIndex or LaserData for semantic search
- **Graph DB** (Amazon Neptune, FalkorDB) for entity resolution — linking the same person across Slack, git, and docs
- **FalkorDB temporal snapshots** for point-in-time queries — "what was the state of auth in March?"

**What you gain over Level 3:**
- Semantic search: "how has our auth approach evolved?" works without keyword matching
- Entity resolution: `@jkim` in Slack, `jkim` in git commits, and `Jake Kim` in a doc are the same person — automatically
- Temporal reasoning: query memory as it existed at any point in time, not just the latest digest
- Cross-program joins: "who has contributed to both auth and platform?" answered from the graph

**The ceiling:** Operational complexity and setup cost. This is real infrastructure. Validate Level 3 first.

**Managed alternative:** [Serro](https://serro.ai) is Level 4 out of the box — proprietary ontology, built-in entity resolution, continuous ingestion with full temporal support. No infrastructure to operate.

**When to add it:** After your Level 3 loop is stable and you're hitting the ceiling of flat digest queries. Don't add it upfront.

```
Loop runs → writes digest → CocoIndex / LaserData → vector index (semantic search)
                                                   → graph DB (entity resolution)
                                                   → temporal snapshots (point-in-time)
```

→ See [CocoIndex](https://github.com/cocoindex-io/cocoindex) · [FalkorDB](https://github.com/FalkorDB/FalkorDB) · [Family C memory store options](family_c/instructions.md#c4--choose-your-memory-store)

---

## Summary

| | Level 1 | Level 2 | Level 3 | Level 4 |
|---|---|---|---|---|
| Setup time | ~20 min | ~2 hours | ~3 hours | Days |
| Ongoing maintenance | None | Mapping file | None | None |
| Query type | Keyword, recent | Keyword, scoped | Keyword, scoped, cached | Semantic, entity-resolved, temporal |
| Context ceiling | Hits fast | Managed | Rarely hit | Not a constraint |
| Infrastructure | None | None | Persistent process | Persistent process + index |
| Right when | Starting out | Stable programs, willing to maintain | Most orgs — start here | Semantic search, entity resolution, or temporal reasoning needed |

**The mapping file (`program_mappings.yaml`) is the same at every level.** Write it once at Level 2 and it carries forward through Level 3, Level 4, and beyond. The loop reads it. CocoIndex reads it. Every Claude session reads it. The investment compounds.
