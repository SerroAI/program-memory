# Comparative Analysis - What We Tried and Why

> Running log of ideas evaluated, what broke, what held up, and the reasoning behind each fork. Read this before picking an implementation.

---

## Idea 1 - MCP-Native Polling

**Date:** 2026-06-08  
**Status:** Partial dead end - works for on-demand queries, structurally broken for continuous ingestion

### What it proposed
Use Claude Code's built-in MCP integrations (GitHub, Slack, Google Drive) with `/schedule` cron jobs to poll for new signals daily and ingest them into a local `org-memory/` directory structure.

### What didn't work

#### ❌ MCP is pull-only - no push, no webhooks
**The assumption:** MCP integrations would receive signals as they happen, keeping program memory continuously up to date.

**The reality:** MCP is request-response. Claude calls the MCP server. The MCP server never calls Claude. There is no event subscription, no webhook listener, no push model anywhere in the MCP protocol.

**Consequence:** "Continuous signal ingestion" is actually polling - Claude wakes up on a schedule and asks "what happened since I last checked?" That means up to 24h signal lag on a daily schedule. Achieving sub-minute latency requires a different architecture - polling can't deliver that without significant infrastructure.

**One way to achieve real-time ingestion:** An always-on backend server with event subscriptions - GitHub webhooks, Slack Events API, Google Drive push notifications - each writing to program memory on arrival. Whether this is what any given product does internally is not publicly known.

#### ❌ No persistent process between Claude sessions
**The assumption:** Claude Code could maintain continuous awareness of org signals.

**The reality:** Claude Code is invoked per-session. When you close the terminal, nothing is running. The `/schedule` skill creates cron jobs that re-invoke Claude on a timer - they don't give Claude persistent awareness between runs.

**Consequence:** Between scheduled runs, a critical decision made in Slack at 3pm is invisible to all agents until 8am the next morning.

#### ❌ Voice updates are a manual shortcut, not a live pipeline
**The assumption:** macOS Shortcut → Claude → memory write would replicate a voice-to-memory feature.

**The reality:** The Shortcut requires deliberate triggering. A production voice integration would require a persistent background listener with immediate propagation - ambient capture, not a keyboard shortcut. Whether any current implementation achieves this reliably is unverified.

#### ⚠️ Collective agent governance is convention-based, not enforced
**The assumption:** CLAUDE.md files referencing org-memory would give all agents shared program context.

**The reality:** This is a social contract. Any Claude session that doesn't load the org-memory CLAUDE.md operates blind. A purpose-built system would make governance architectural - agents connect to a memory store as a hard dependency rather than optionally reading a config file.

### What held up

#### ✅ Directory structure + CLAUDE.md conventions
The `programs/<name>/` structure with `charter.md`, `memory.md`, and `signals/YYYY-MM.md` is a solid data model. Human-readable, git-diffable, portable. Worth keeping as the storage layer regardless of how signals get written.

#### ✅ MCP tools for on-demand queries
When Claude is active in a session, MCP integrations work well for ad-hoc lookups. Pull model is fine for interactive queries - it only breaks for continuous ingestion.

#### ✅ Temporal symbol history via git + program cross-reference
`git log -p -S "<symbol>"` + cross-referencing commit dates against signals files is valid. Doesn't require real-time ingestion - reads historical git data.

#### ✅ On-demand analyses
Reading accumulated signals files and running a summarization workflow works. The analytical layer is sound; quality depends on signal coverage.

---

## The Deeper Question - Do We Need to Ingest at All?

**2026-06-08** - Before designing a better ingestion pipeline, challenge the premise.

The MCPs already have the latest data. GitHub has every PR. Slack has every message. Drive has every doc. If Claude can query them on demand - why copy anything?

**Ingestion is only justified if two problems are real:**

### Problem 1 - Cross-source joins
The MCPs are siloed. GitHub doesn't know about Slack. Slack doesn't know what a "program" is. Queries that require answers spanning multiple sources - "what's blocking enterprise-readiness across GitHub + Slack + Drive?" - need something to do the join. The question is when: at query time (live synthesis) or ahead of time (ingestion).

**Don't assume this is needed until a real use case demands it.**

### Problem 2 - Context window cost at scale
Querying GitHub + Slack + Drive in parallel means multiple large raw API responses. For small orgs or scoped queries this is fine. For quarterly analyses across 5 programs and 10 contributors it becomes expensive and slow. Pre-summarized memory trades write cost for read cost.

**Don't pre-optimize. Measure a real query load first.**

---

## The Fork

Two families of solutions emerge from this question:

**Family B - Human-maintained source mapping**  
Each program declares which sources belong to it. Claude queries only those declared sources at query time. Simple. Zero infrastructure. Requires human compliance to keep the mapping updated. See [`family_b_source_mapping.md`](family_b_source_mapping.md).

**Family C - Auto-ingestion**  
Signals are discovered and written to memory automatically. No human maintains a mapping. Broader coverage. More infrastructure. Three implementation paths - see [`family_c/`](family_c/).

Neither is wrong. They serve different trust models. Family B trades coverage for simplicity. Family C trades simplicity for completeness.

---

## Open Questions

- Cross-source joins: do we have a real use case that requires them, or is it hypothetical? Build a query and measure before committing to ingestion.
- Context window cost: what does a full 5-program quarterly synthesis actually cost in tokens?
- Can the webhook server be a Claude Code hook (`settings.json` hooks) rather than a separate process?
- Does Google Meet have a push API for transcripts, or do they only land in Drive after the meeting ends?
- Could Claude.ai Projects serve as the shared memory layer natively, eliminating local files entirely?
- Vector store instead of markdown files - worth it for semantic retrieval?
