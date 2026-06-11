# Key Decisions - What We Learned and Why It Matters

> 13 decision points discovered through the design process. Each one has real consequences for architecture, complexity, and whether the system works at all. Ordered by when they appear in the build journey.

For the full interactive decision tree, see [`../memory_layer_decision_chart.md`](../memory_layer_decision_chart.md).

---

## Memory Layer Decisions

### 1. Long-horizon reasoning needed?
**The decision:** Do you need to ask questions that reason across time? - "how has scope drifted over 3 months", "what did we spend time on in Q3 vs Q4", "what decisions changed direction mid-quarter."

**Why it matters:** This fork has two distinct dimensions that are often conflated:

**Dimension A - History depth.** Slack Free/Pro limits message history to 90 days (hard platform limit, not an API constraint). Slack Business+/Enterprise has full history. GitHub's full commit history is reachable via MCP but responses are paginated: 30 results per call by default, max 100 per call, requires incrementing the `page` parameter for additional results. For long time windows with many commits, multi-page querying is required.

**Dimension B - Search type (more significant).** GitHub MCP searches commit messages and file diffs by keyword match (grep-style). There is no embedding or semantic similarity search in the GitHub API or its MCP wrapper. A question like "what changed about our authentication approach over 6 months" only surfaces commits where the word "authentication" (or your keyword) appears literally. Semantic queries - finding conceptually related changes that use different terminology - cannot be answered by keyword search alone, regardless of history depth. This gap cannot be worked around within Family B; it requires ingestion with an embedding index (Family C).

**Decision outcome:**
- Need semantic long-horizon reasoning → Family C required (embedding index needed)
- Need deep history, keyword queries only, Slack Business+ → Family B viable with multi-page GitHub querying
- Unsure → test a keyword temporal question against live APIs; note where answers feel incomplete

---

### 2. Org size?
**The decision:** How many engineers, how many active programs?

**Why it matters:** Family B fires N × M parallel MCP calls per query (programs × sources per program). At small scale (≤15 engineers, 2–3 programs), total signal volume likely fits in Claude's context window. At larger scale, context window cost and API rate limit risk both increase. This is not a hard cutoff - it's a risk flag that triggers a context window test before committing.

**Decision outcome:**
- Small org → proceed to discipline check, then run context window test
- Larger org → flag the risk, still run the test, consider Family C if test fails

---

### 3. Team discipline for mapping maintenance?
**The decision:** Is your team willing and able to update `program_mappings.yaml` when channels are renamed, repos archived, or meetings change - roughly once a quarter?

**Why it matters:** Family B's failure mode is silent. When a source is missing from the mapping, the system returns a complete-looking answer that is invisibly incomplete. No warning, no error. A renamed Slack channel that nobody updates in the mapping means every query silently excludes all messages in that channel - indefinitely. Teams that haven't experienced this kind of silent data gap tend to underestimate how damaging it is in practice.

**Decision outcome:**
- Yes, team owns it → proceed
- No / unreliable → Family C. Silent degradation is worse than the complexity of ingestion.

---

### 4. Accept silent gaps?
**The decision:** Even with a disciplined team, do you explicitly accept that unmapped sources will be invisible to the system?

**Why it matters:** This is a separate question from discipline. A disciplined team still has moments where a new channel gets created and the mapping update happens two weeks later. During those two weeks, every query against that program silently misses everything in that channel. The question is whether the team understands and accepts this as an explicit, known tradeoff - not something that will "get fixed eventually."

**Decision outcome:**
- Yes, explicitly accepted → proceed to rate limit check
- No, need full coverage → Family C

---

### 5. Query frequency / rate limit risk?
**The decision:** How often will queries be fired against the declared sources?

**Why it matters:** Family B pays API cost per query. Family C pays API cost once per event (at ingestion time). With low query frequency (a few times a day), rate limits are unlikely to be a problem. With high query frequency (multiple users querying throughout the day, automated agents querying on every task), Slack, GitHub, and Drive APIs can all hit rate limits. This is a risk flag, not a hard block - but it should inform the decision.

**Decision outcome:**
- Low frequency → proceed to context window test
- High frequency → flag the risk; consider Family C to amortize API cost across ingestion

---

### 6. Context window test passed?
**The decision:** Before committing to Family B, run a real cross-source query and measure whether it works at your actual scale.

**Why it matters:** This is the only empirical gate in the Family B path. Everything before it is reasoning about risk; this is measurement. The test: fire a cross-source query against all declared sources for one program, measure total tokens returned, check whether the answer fits in context without truncation, measure end-to-end latency, and evaluate answer quality against a human ground truth. If it fails at your current scale, it will fail worse as the org grows.

**Decision outcome:**
- Passed → Family B is viable now. Re-run as org grows.
- Failed → Family C. The failure tells you what dimension to optimize for (context size, latency, or answer quality).

---

## Family C Option Decisions

### 7. Latency tolerance?
**The decision:** How stale can program memory be when a query is answered?

**Why it matters:** The three Family C options differ primarily on latency. C2 (git + cron) accepts up to 60-minute lag. C3 (GitHub Actions + Cloudflare Worker) achieves 1–2 minutes. C1 (webhook server) achieves seconds. The right choice depends on whether hourly staleness is acceptable for your use cases - for most program queries ("what's the state of this program?"), hourly is fine. For real-time incident response or immediate follow-up on meeting decisions, it's not.

**Decision outcome:**
- Hours acceptable → C2 (start here, lowest complexity)
- Minutes acceptable → C3 (upgrade when lag becomes a real problem)
- Seconds required → C1 (only if willing to operate infrastructure)

---

### 8. Willingness to operate infrastructure?
**The decision:** Are you willing to run an always-on server 24/7 with uptime monitoring, restarts, and webhook signature validation?

**Why it matters:** C1 requires a server you operate. C3 uses GitHub Actions (GitHub operates it) with a stateless Cloudflare Worker (free tier, no uptime management). If the answer to C1 is no but you need near-real-time ingestion, C3 is the right fallback. The Cloudflare Worker is the only code you own in C3 - approximately 10 lines.

**Decision outcome:**
- Yes → C1 viable
- No → C3 is the near-real-time option without server operations

---

---

## Out of Scope — Proactive, Widget, and Agent Governance Layers

> Decisions 9–13 are documented here for completeness, but they are **not in scope for this repo**. This repo covers the memory layer only (decisions 1–8). The proactive layer, action item follow-through, and widget layer require validated memory as a prerequisite and are not implemented here.
>
> If you need production-grade versions of these capabilities without building them yourself, see [Serro](https://serro.ai).

---

### 9. Memory layer validated before proceeding?
**The decision:** Before building the proactive monitoring layer, have you measured memory quality: classification accuracy, signal coverage, and staleness lag?

**Why it matters:** This is a hard gate. Proactive alerts built on top of poorly classified memory are noisy and wrong - they surface false risks and miss real ones. Noisy alerts train teams to ignore the system, which is worse than no alerts at all. The memory layer needs to demonstrate acceptable classification accuracy (suggested minimum: >80% on a human-labeled sample) before the proactive layer is worth building. 30 days of real operation on a real org is the minimum measurement window.

**Decision outcome:**
- Not yet validated → do not proceed. Run the measurement first.
- Validated and acceptable → proceed to proactive layer design

---

### 10. Proactive monitoring needed + always-on process available?
**The decision:** Do you need the system to alert proactively on program drift, stalled items, and silent contributors - and do you have an always-on process to run the monitoring loop?

**Why it matters:** Proactive monitoring requires a persistent process that watches program state and initiates alerts without being asked. Claude Code sessions are reactive - they answer when called. C2's cron and C3's scheduled Actions approximate proactive monitoring within a polling window. Family B has no trigger mechanism at all. The question is whether the polling latency (up to 60 minutes for C2) is acceptable for your monitoring use case.

**Decision outcome:**
- Needed + always-on available → build proactive agent
- Needed + Family B → accept latency limitations or upgrade to C2/C3
- Not needed → skip to action item decision

---

### 11. Action item extraction accuracy tested?
**The decision:** Before shipping action item follow-through, have you measured extraction precision on real meeting transcripts and Slack threads?

**Why it matters:** Action item extraction from unstructured text has real error rates. False positives - the system follows up on something that wasn't a commitment - erode trust faster than false negatives. A suggested precision floor before shipping is 85%: at that level, most follow-ups are legitimate and the system feels useful. Below that, follow-ups feel random and teams start ignoring them. Test on 20 real samples before shipping.

**Decision outcome:**
- Not tested → run the measurement before shipping
- Tested and acceptable → proceed
- Tested and below threshold → improve extraction before shipping; do not ship noisy follow-ups

---

## Widget Layer Decisions

### 12. Memory live and validated before widgets?
**The decision:** Is the memory layer running, accurate, and validated before building the widget layer?

**Why it matters:** Widgets are a display layer. A widget configured to show "open blockers for enterprise-readiness" is only useful if the memory layer contains accurate, up-to-date signal data about blockers. A widget querying empty or miscategorized memory returns a confident-looking answer that is wrong. This is a hard block - there is no version of the widget layer that works without validated memory behind it.

**Decision outcome:**
- Memory not validated → hard block. Build and validate memory first.
- Memory validated → proceed

---

### 13. Engineering capacity for full-stack widgets?
**The decision:** Do you have the capacity to build a full-stack application - backend API, Claude API proxy, real-time update mechanism, structured output schemas, and a frontend renderer?

**Why it matters:** Widgets are not a thin UI. They require: a backend server (the frontend cannot read git directly), a Claude API proxy (API keys cannot be exposed in the browser), a mechanism to detect memory updates and trigger re-execution (polling or WebSocket/SSE), structured output schemas to render consistently across executions, and context management as memory grows. This is a meaningful engineering investment. If capacity is limited, a Slack digest is a viable lightweight substitute: a scheduled agent summarizes program state and posts to a channel - no frontend, no backend, no real-time infrastructure required.

**Decision outcome:**
- Capacity available → build widget layer
- Capacity limited → Slack digest as fallback (scheduled agent → Slack MCP → channel post)
