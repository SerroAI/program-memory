# Capability Analysis - Live Program Memory Layer

> This document defines the capabilities of the memory layer we are building. Scope is limited to the live shared program memory layer — signal ingestion, program-indexed storage, and queryable retrieval. The proactive coordination layer, contributor graph, and widget layer are out of scope for this repo.
>
> **Epistemics:** Difficulty ratings and open questions are engineering judgments, not measured findings. Nothing here has been tested against a real org. Treat this as a working hypothesis.

---

## In Scope: Memory Layer

| # | Capability | Est. replication difficulty | Can Claude do it today? | Key gap (unvalidated) |
|---|---|---|---|---|
| 1 | Program-structured memory | Medium | Partial | Auto-curation unbuilt |
| 2 | Continuous signal ingestion | High | Partial | No auto-classification or dedup |

**Measurement dimensions for every capability:** Accuracy · Coverage · Latency/Staleness

---

## Capability Detail

---

### 1. Program-Structured Memory

**What it does:** Organizational signals — decisions, code changes, meetings, threads — are indexed to a named program rather than stored flat. Retrievable by program context.

**Why this matters:** Flat memory stores require humans to do the joining. Program-indexed memory lets a query like "what's the state of Enterprise Readiness?" return a scoped answer. Whether this framing is meaningfully better than tagging or folder structure is an empirical question, not a given.

**Replication difficulty:** Medium. The concept is straightforward. The difficulty is auto-curation — getting signals classified correctly into programs without human filing. Without that, program-structured memory is just a filing convention that requires discipline to maintain.

**Open questions:**
- How accurate is auto-classification in practice? Unvalidated.
- Does program-structured memory produce better outcomes than well-tagged flat memory? Unvalidated.

**Measure:**
- Precision: % of retrieved facts that belong to the queried program (vs. noise)
- Recall: % of actual program events captured vs. ground truth log
- Staleness: median lag between a real work signal and it appearing in program memory

---

### 2. Continuous Signal Ingestion

**What it does:** Signals from GitHub, Slack, and Google Drive auto-classify into the correct program's memory without human filing.

**Why this matters:** Manual curation of a program wiki typically degrades within weeks. Automatic ingestion, if it works reliably, removes the maintenance burden. The word "if" is load-bearing — classification accuracy, deduplication, and handling of ambiguous signals are unsolved engineering problems.

**Replication difficulty:** High. Requires reliable integrations, a classification step, and deduplication. Building a version that works is tractable; building one that works well is harder.

**Open questions:**
- What is actual classification accuracy at scale? Requires a real org to measure.
- How does the system handle signals that belong to multiple programs?
- What is the deduplication strategy when the same event appears in GitHub and Slack?

**Measure:**
- Auto-classification accuracy: % of signals routed to correct program (human-labeled sample)
- Coverage: ratio of signals captured vs. signals emitted per integration
- Duplicate rate: % of events stored more than once

---

## Out of Scope

The following capabilities depend on the memory layer being stable first and are not addressed in this repo:

- **Temporal code symbol understanding** — requires classification from capability 2 to work correctly
- **Contributor expertise graph** — requires cross-signal identity resolution across tools
- **Voice-driven memory updates** — requires a persistent background capture process
- **Collective agent governance** — requires a persistent shared state architecture beyond CLAUDE.md conventions
- **Large-scale org analyses** — quality is bounded by signal coverage from capabilities 1 and 2
- **Program health monitoring and proactive alerts** — requires an always-on process and a defined model of program health
- **Action item follow-through** — requires persistent state and outbound messaging
- **Live prompt-based widgets** — full-stack application, depends on all of the above

**Build order:** Validate capabilities 1 and 2 first. Everything else is downstream of them working correctly.

---

## What Would Change This Analysis

Running capabilities 1 and 2 against a real org for 30 days and measuring classification accuracy, coverage, and staleness. Everything above is hypothesis until then.
