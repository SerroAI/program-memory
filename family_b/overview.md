# Family B - Programs-to-Sources Mapping

> Human-maintained. Each program declares its sources. Claude queries only those sources at query time. No ingestion pipeline.

---

## The Idea

Each program has a declared set of sources in `programs_to_sources_mapping.yaml`. When a question is asked about a program, Claude reads the mapping and fires targeted MCP calls against only the listed sources - no blind org-wide scanning.

```yaml
# programs_to_sources_mapping.yaml
enterprise-readiness:
  slack_channels:
    - "#enterprise-readiness"
    - "#sales-eng"
    - "#security"
  github_repos:
    - auth-service
    - billing-service
    - admin-portal
  drive_folders:
    - "Programs/Enterprise Readiness"
  recurring_meetings:
    - "Weekly Enterprise Sync"

platform-migration:
  slack_channels:
    - "#platform"
    - "#infra"
  github_repos:
    - platform-core
    - legacy-api
  drive_folders:
    - "Programs/Platform Migration"
  recurring_meetings:
    - "Platform Migration Standup"
```

## Why It Works

- Cross-source joins are solved: Claude knows exactly where to look per program
- Always-live data - MCPs return current state on every query
- Zero ingestion infrastructure - no cron, no server, no webhook
- The only thing maintained manually is the mapping itself - changes maybe once a quarter

## The Accepted Tradeoff

> ⚠️ This approach requires human compliance on two fronts:
> 1. Program owners keep the mapping updated as channels, repos, and meetings change
> 2. Downstream agents always read the mapping before querying rather than going wide
>
> Work that happens outside declared sources is **silently** excluded - no warning, no error. A new Slack channel spun up for a program that nobody adds to the mapping means every query against that program misses it indefinitely.
>
> This is acceptable if the team explicitly owns the mapping as a lightweight operational responsibility. It is not acceptable if the expectation is zero-maintenance coverage.

## Known Limitations of Family B

These are unvalidated hypotheses - each one should be tested before treating it as a confirmed reason to prefer Family C.

### 1. Long-horizon reasoning requires preprocessing
Family B queries are always live - they return the current state of declared sources. There is no accumulated record of past states. This means Family B likely cannot support queries that require reasoning across time: "how has the scope of this program drifted over the last 3 months?", "what decisions changed direction mid-quarter?", "where did we spend most of our time in Q3 vs. Q4?" - all of these require a historical record, not a live snapshot.

Without some form of preprocessing (ingestion, even lightweight), temporal and long-horizon reasoning is not possible regardless of context window size. **This may be the most significant structural limitation of Family B.**

Flag: needs a test. Ask Family B a long-horizon question against a real org and measure whether the live APIs can supply enough historical data to answer it.

### 2. Context window cost - possibly a non-problem at small org scale
Querying GitHub + Slack + Drive simultaneously returns raw API payloads that could be large. But for a small company (≤15 engineers, 2–3 programs, a few months of history), total live signal volume might fit comfortably in Claude's context window - making ingestion unnecessary for cross-source joins at that scale.

**This is unvalidated.** Proposed test: run a Family B cross-source query against a real small org, measure total tokens returned, whether it fits in context, latency, and answer quality. Context window cost may only become a reason to consider ingestion as the org grows.

### 3. API rate limits
Every query fires multiple MCP calls in parallel. Slack, GitHub, and Drive all have rate limits. High query frequency - multiple team members querying throughout the day - could hit them. Ingestion pays the API cost once per event; Family B pays it on every query.

### 4. Historical depth is bounded by API limits
Slack free tier limits search to 90 days. GitHub API paginates and may miss older commits on large repos. Drive search is not semantic or exhaustive. Family B can only answer questions about what the APIs currently expose - not a full historical record.

### 5. The mapping is a silent single point of failure
If the mapping is wrong - a renamed channel, an archived repo, a new meeting series nobody added - every query returns incomplete results with no warning. There is no self-correction mechanism. The system cannot tell the difference between "nothing happened in this channel" and "this channel doesn't exist in the mapping."

### 6. Query cost scales with program count
A cross-program query fires N × M MCP calls (programs × sources per program). At 5 programs × 5 sources, that is 25 API calls per query, each returning raw content Claude must synthesize. Cost and latency grow with org complexity.

---

## Where the Options Diverge

Family B and Family C are not better/worse - they reflect different assumptions about the team:

| | Family B | Family C |
|---|---|---|
| Who maintains coverage? | Humans (explicit config) | System (auto-discovery) |
| What happens when a new channel is created? | Silent gap until mapping is updated | Automatically ingested |
| Infrastructure required | None | Cron job at minimum |
| Data freshness | Always live | Depends on ingestion latency |
| Right for | Disciplined teams, small orgs, getting started | Teams that want automatic coverage |

## Implementation

See [`../family_b/instructions.md`](../family_b/instructions.md) for full setup steps.
