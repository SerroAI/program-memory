# Capability Analysis - AI Technical Program Manager

> This document maps the capabilities of an AI TPM (Technical Program Manager) system as described by Serro's public blog and founder statements. These are the capabilities we are attempting to replicate or match using Claude Code and open-source tooling.
>
> **Epistemics:** Every claim about Serro's internal implementation is inferred from public information, not verified. Every "replication difficulty" rating is a hypothesis based on engineering judgment, not a measured finding. Nothing here has been tested against a real org. Treat this as a working hypothesis, not a conclusion.

---

## The Three Layers

| Layer | What it does | Passive or Active? |
|---|---|---|
| **Live Shared Program Memory** | Ingests signals, organizes by program, keeps context queryable | Passive - responds to signals |
| **Proactive Program Coordinator Agent** | Monitors program state, alerts on drift, follows up on action items | Active - initiates without being asked |
| **Prompt-based widgets** | User-configured views that re-execute against memory on a refresh interval | Active - surfaces state without requiring a query |

Each layer depends on the one above it. Memory without ingestion is empty. The proactive agent without memory has no context. Widgets without live memory return meaningless output.

---

## TL;DR - 10 Capabilities

**Live Shared Program Memory layer:**

| # | Capability | Est. replication difficulty | Can Claude do it today? | Key gap (unvalidated) |
|---|---|---|---|---|
| 1 | Program-structured memory | Medium | Partial | Auto-curation unbuilt |
| 2 | Continuous signal ingestion | High | Partial | No auto-classification or dedup |
| 3 | Temporal code symbol understanding | High | Partial | No program context layer on top of git |
| 4 | Contributor expertise graph | High | Partial | No cross-signal join |
| 5 | Voice-driven live memory updates | Medium | No | Full gap - untested |
| 6 | Collective agent governance | High | No | Claude is session-scoped |
| 7 | Large-scale org analyses | Medium | Partial | Quality depends on ingestion working first |

**Proactive Program Coordinator layer:**

| # | Capability | Est. replication difficulty | Can Claude do it today? | Key gap (unvalidated) |
|---|---|---|---|---|
| 8 | Program health monitoring + proactive alerts | High | No | Requires always-on process |
| 9 | Action item follow-through | High | No | Requires persistent state + outbound messaging |

**Widget layer:**

| # | Capability | Est. replication difficulty | Can Claude do it today? | Key gap (unvalidated) |
|---|---|---|---|---|
| 10 | Live prompt-based widgets | High | No | Full-stack application - depends on memory layer existing first |

**Measurement dimensions for every capability:** Accuracy · Coverage · Latency/Staleness

---

## Capability Detail

---

### 1. Program-Structured Memory
**What it does (described):** Organizational signals - decisions, code changes, meetings, tickets, threads - are indexed to a named program rather than stored flat. Retrievable by program context.

**Why this might matter:** Flat memory stores (Notion, Confluence, raw search) require humans to do the joining. Program-indexed memory lets a query like "what's the state of Enterprise Readiness?" return a scoped answer. Whether this framing is meaningfully better than tagging or folder structure is an empirical question, not a given.

**Replication difficulty:** Medium. The concept is straightforward. The difficulty is auto-curation - getting signals classified correctly into programs without human filing. Without that, program-structured memory is just a filing convention that requires discipline to maintain.

**Open questions:**
- How accurate is Serro's auto-classification in practice? We don't know.
- Does program-structured memory produce better outcomes than well-tagged flat memory? Unvalidated.

**Measure:**
- Precision: % of retrieved facts that belong to the queried program (vs. noise)
- Recall: % of actual program events captured vs. ground truth log
- Staleness: median lag between a real work signal and it appearing in program memory

---

### 2. Continuous Signal Ingestion & Auto-Curation
**What it does (described):** Signals from GitHub, Jira, Slack, and meeting transcripts auto-classify into the correct program's memory without human filing.

**Why this might matter:** Manual curation of a program wiki typically degrades within weeks. Automatic ingestion, if it works reliably, removes the maintenance burden. The word "if" is load-bearing - classification accuracy, deduplication, and handling of ambiguous signals are unsolved engineering problems.

**Replication difficulty:** High. Requires reliable integrations, a classification step, and deduplication. Whether Serro's classification is accurate at scale is unknown from public information. Building a version that works is tractable; building one that works well is harder.

**Open questions:**
- What is Serro's actual classification accuracy? Not publicly disclosed.
- How does the system handle signals that belong to multiple programs?
- What is the deduplication strategy when the same event appears in GitHub and Slack?

**Measure:**
- Auto-classification accuracy: % of signals routed to correct program (human-labeled sample)
- Coverage: ratio of signals captured vs. signals emitted per integration
- Duplicate rate: % of events stored more than once

---

### 3. Temporal Code Symbol Understanding
**What it does (described):** A time-indexed map of code symbols - who created or modified each one, when, in which sprint, for which program.

**Why this might matter:** `git blame` gives author and date. Program context layered on top could answer "why was this function changed" rather than just "who changed it." The value depends on classification accuracy from capability 2 - if signals aren't correctly attributed to programs, the program context layer is wrong.

**Replication difficulty:** High. `git log` and `git blame` give the raw data. The hard part is reliably joining commit history to program assignments, especially for commits that predate the program taxonomy or touch multiple programs. We do not know how well a production system actually achieves this.

**Open questions:**
- How does a system handle commits that span multiple programs?
- How far back does temporal understanding reliably work?
- Is this capability demonstrably useful compared to `git log` + good commit messages?

**Measure:**
- Symbol attribution accuracy: % of symbols correctly attributed to a program (sampled, human-verified)
- Temporal resolution: can correctly answer "what changed in this symbol during a given time window?"
- Cross-reference depth: # of context layers available per symbol (commit → PR → program)

---

### 4. Contributor Expertise Graph
**What it does (described):** A cross-signal map of who has worked on what - by program, by code area, by time period - joined across GitHub, Slack, and docs.

**Why this might matter:** Reviewer selection, escalation, and onboarding decisions benefit from knowing who has context in an area. Any single source (GitHub commits, Jira assignees) is a partial picture. Cross-source joining produces a more complete one - assuming the join is accurate.

**Replication difficulty:** High. The join across GitHub + Slack + docs requires identity resolution (mapping GitHub handles to Slack handles) and weighting decisions (is a Slack message worth as much as a commit?). The quality of the resulting graph is sensitive to these choices and hard to validate without ground truth.

**Open questions:**
- How does identity resolution work across systems with different handles?
- How is expertise weighted - recency vs. volume vs. depth?
- Does this graph produce demonstrably better reviewer recommendations than GitHub's built-in suggestions?

**Measure:**
- Recall of top-N contributors for a code area vs. ground truth (human eval)
- Attribution freshness: does the graph reflect recent activity accurately?
- Cross-signal join rate: % of contributors appearing in ≥2 signal sources

---

### 5. Voice-Driven Live Memory Updates
**What it does (described):** Program owners update program memory via voice in real time. The update propagates to memory immediately.

**Why this might matter:** The moment of context - a decision made in a meeting - is often not at a desk. If capture requires opening a tool, it frequently doesn't happen.

**Replication difficulty:** Medium. Voice transcription (Whisper or equivalent) is commodity. The hard parts are: structured extraction from free-form speech (deciding what to write to memory and where), accuracy (no hallucinated scope changes), and propagation latency. Whether any production system handles these reliably is unverified.

**Open questions:**
- What is the false-positive rate for voice-extracted decisions? (Capturing something that wasn't a decision)
- What is the false-negative rate? (Missing a decision that was stated)
- Does this work in noisy environments or with multiple speakers?

**Measure:**
- Capture rate: % of stated decisions that appear correctly in memory
- Latency: time from voice utterance to memory availability
- Accuracy: % of extractions that match human-verified intent

---

### 6. Collective Agent Governance via Shared Program Memory
**What it does (described):** Downstream agents - code reviewer, sprint planner, onboarding agent - read the same live program memory as shared context rather than operating with isolated per-session state.

**Why this might matter:** Per-session stateless agents can produce individually good but collectively incoherent outputs. Shared memory could give a fleet of agents consistent context. Whether this produces meaningfully better outcomes than well-prompted individual agents is an empirical question.

**Replication difficulty:** High. Claude is session-scoped. A CLAUDE.md convention provides shared instructions but not shared state. Genuine shared memory requires an architecture decision - a persistent store that multiple agents read from, with a defined update model and permissions layer.

**Open questions:**
- Is collective governance meaningfully better than per-session prompting in practice?
- What happens when two agents update memory simultaneously?
- How does permissions work - can all agents read all programs?

**Measure:**
- Context consistency: do two agents querying the same program at the same time return consistent facts?
- Governance coverage: % of agent calls that actually read shared memory vs. operating stateless
- Propagation correctness: when memory updates, how long before all agents reflect the new state?

---

### 7. Large-Scale Org Analyses
**What it does (described):** On-demand rollups - "what has engineering spent time on this quarter, by program?" or "where has R&D investment gone?" - across the full signal corpus.

**Why this might matter:** Leadership decisions about headcount and investment require this kind of data. Currently it either doesn't exist or requires bespoke dashboards. An on-demand query layer is faster.

**Replication difficulty:** Medium for the query layer; high for the corpus quality it depends on. If the signal corpus is sparse or miscategorized, the analysis is wrong regardless of how good the query layer is. This capability is downstream of capabilities 1 and 2 working correctly.

**Open questions:**
- How accurate is effort attribution when signal classification has errors?
- How does the system handle work that wasn't captured in any integrated signal source?
- What does a quarterly analysis actually look like - does it produce actionable output or noise?

**Measure:**
- Coverage: % of real work effort captured vs. estimated ground truth
- Accuracy: correlation with human-audited time logs (sampled)
- Latency: time to generate a full quarterly analysis

---

### 8. Program Health Monitoring + Proactive Alerts
**What it does (described):** The system continuously monitors program state against the program's charter. Without being asked, it surfaces programs drifting from scope, stalled action items, dependencies at risk, contributors who've gone silent.

**Why this might matter:** A passive memory store requires someone to query it. Proactive monitoring removes that requirement - the system tells you when something is wrong rather than waiting to be asked.

**Replication difficulty:** High. Requires an always-on process, a defined notion of "drift" or "at risk" that can be evaluated programmatically, and outbound communication capability. Currently approximated with scheduled agents — a polling workaround, not a native capability. Cloud workflows with persistent loops would close this gap and turn program health monitoring into a first-class primitive. Defining what "at risk" means in a way that produces useful alerts without noise is a product problem, not just an engineering one.

**Open questions:**
- What is the false positive rate on proactive alerts? Noisy alerts get ignored.
- How is "drift from scope" defined in a way an automated system can detect?
- Does this require human tuning per program, or does it work out of the box?

**Measure:**
- Alert precision: % of proactive alerts representing real risks
- Alert recall: % of actual risks surfaced before a human noticed
- Lead time: days of advance notice before human escalation

---

### 9. Action Item Follow-Through
**What it does (described):** When an action item is assigned in a meeting or thread, the system tracks it. If it hasn't moved by the expected date, the system follows up directly with the owner in context.

**Why this might matter:** The most common failure mode in coordination is not missed decisions but decisions that were made, assigned, and then silently forgotten. Follow-through automation addresses this.

**Replication difficulty:** High. Requires: extracting action items accurately from unstructured signal (meetings, Slack), associating them with owners and due dates, tracking state over time, and generating follow-up messages that are useful rather than annoying. Each step has meaningful error rates. The state persistence and follow-up scheduling problems are currently worked around with cron-triggered agents; cloud workflows with loops would make this architecture significantly cleaner.

**Open questions:**
- How accurate is action item extraction from meeting transcripts? Transcripts are noisy.
- How does the system determine that an action item "hasn't moved"?
- What is the threshold between helpful follow-up and annoyance?

**Measure:**
- Extraction accuracy: % of real action items correctly identified from source signals
- Closure rate: % of tracked items that reach a resolved state
- Annoyance rate: % of follow-ups reported as redundant or unwanted

---

### 10. Live Prompt-Based Widgets
**What it does (described):** Users configure views by writing a prompt. The widget executes that prompt against live program memory on a refresh interval and displays the result without requiring a query.

**Why this might matter:** Program state made ambient - visible in a sidebar or dashboard - without requiring anyone to ask. The "prompt-based" distinction means any view is configurable in natural language, not by a data team.

**Critical dependency:** This capability is entirely contingent on capabilities 1 and 2 working correctly. A widget querying empty or miscategorized memory returns meaningless output. Do not design or build this until the memory layer is validated.

**Replication difficulty:** High. Requires a backend API (frontend cannot read git), a Claude API proxy, a mechanism to detect memory updates and trigger re-execution, structured output schemas for consistent rendering, and context management for large memory stores. This is a full-stack application, not a thin UI layer.

**Open questions:**
- How does structured output work across widget types with different schemas?
- What is the rendering strategy when Claude output format varies between executions?
- How does the system bound context window size as memory grows?

**Measure:**
- Widget freshness: lag between memory update and widget reflecting it
- Prompt fidelity: % of outputs that correctly answer the configured prompt
- Configuration time: how long for a non-technical user to create a useful widget

---

## What the Analysis Suggests

The Live Shared Program Memory layer (1–2) is a prerequisite for everything else. Quality of the whole system is bounded by classification accuracy and signal coverage - two things we have not measured.

The proactive TPM layer (8–9) is currently constrained by Claude's session-scoped execution model — a scheduled agent approximates it but is not the same thing. This is likely a temporary gap. Cloud workflows with persistent loops would make always-on program monitoring a first-class primitive rather than a polling workaround. If that lands, the combination of Serro-style program-indexed memory and looping cloud agents becomes a genuinely compelling architecture — not a replica of Serro, but arguably a stronger version of it: open, composable, and running natively in the tools engineers already use.

The widget layer (10) is last and depends on everything before it.

**Build order:**
1. Memory structure + signal ingestion - together, as one layer
2. Reactive queries against that memory
3. Proactive monitoring via scheduled agent
4. Action item tracking and follow-through
5. Widget layer

**What would change this analysis:** Running capability 1 and 2 against a real org for 30 days and measuring classification accuracy, coverage, and staleness. Everything above is hypothesis until then.
