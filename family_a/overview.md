# Family A: Full Context Pull — Overview

> The simplest possible implementation. No mapping file, no ingestion, no infrastructure. Claude pulls all sources at query time and synthesizes across them in a single context window.

---

## How it works

When you ask Claude a program-level question, it:

1. Reads your `CLAUDE.md` to find the list of sources for this org
2. Queries all of them simultaneously via MCP — GitHub, Slack, Drive, meetings
3. Synthesizes the answer in one context window

There is no pre-processing, no stored state, and no configuration beyond a list of sources. Everything is live.

---

## When to use it

Family A is the right choice when your total source count is small enough to fit in a single context window:

- ≤5 engineers
- 1–2 repos
- 1–3 Slack channels
- 1–2 active programs
- ≤20 sources total

If you're not sure whether you qualify, try it. The context window test in [A4](instructions.md#a4--run-your-first-query) will tell you quickly.

---

## What it costs

| Dimension | Cost |
|---|---|
| Setup time | 30–60 minutes |
| Infrastructure | None |
| Maintenance | Keep `CLAUDE.md` source list current as org changes |
| Query latency | 10–30 seconds per cross-source query |
| Scalability | Re-test as org grows. Upgrade to Family B when sources exceed context window |

---

## Known limitations

**Doesn't scale.** As repos, channels, and programs multiply, context windows fill up and answers degrade. Family A is a starting point, not a destination.

**No semantic search.** Claude can only match on keywords it finds in the pulled content. Questions like "how has our auth approach evolved conceptually" require embeddings — see Family C.

**Latency grows with sources.** Every query pulls from every source. As source count grows, latency grows.

**No history beyond MCP limits.** GitHub MCP returns recent commits (paginated). Slack free/pro plans have a 90-day message limit. Family A is bounded by what the MCP tools can return.

---

## Files in this folder

- [`instructions.md`](instructions.md) — step-by-step setup (A1–A5)

---

## Next step

→ [`instructions.md`](instructions.md)
