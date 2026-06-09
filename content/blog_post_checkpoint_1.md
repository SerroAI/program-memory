# Blog Post — Checkpoint 1
## Building a Serro Replica with Claude Code: What We Learned in Session One

> Status: draft. Tone: technical but accessible. Audience: engineering leaders and senior engineers who know what AI-native tooling is but haven't gone deep on the architecture.

---

There's a startup called Serro building what they call "living organizational memory for AI-native engineering teams." The pitch: every signal from your GitHub, Slack, meetings, and docs — automatically organized by program, continuously updated, and available to every AI agent in your org as shared context.

It's a compelling idea. And I wanted to know: how much of it can you replicate using only Claude Code and its native integrations?

This post is the first checkpoint. We haven't built anything yet. What we've done is map the capabilities, identify the real architectural problems, and land on a set of candidate approaches. That's the work you have to do before you write any code.

---

## What Serro Actually Does

Before you can replicate something you have to understand it precisely. Not the marketing copy — the mechanics. After reading Serro's blog and working through their product, we landed on seven core capabilities:

**1. Program-structured memory.** Every signal — commits, Slack threads, meeting decisions, docs — is indexed to a named *program*. A program is a sustained, cross-functional commitment to an outcome: Enterprise Readiness, Platform Migration, Discoverability. Programs are the unit of memory, not dates or repos or people.

**2. Continuous signal ingestion.** Signals flow in from GitHub, Jira, Slack, and meeting transcripts automatically. No human files anything. The memory curates itself.

**3. Temporal code symbol understanding.** Serro maintains a time-indexed map of every code symbol — who created it, who modified it, in which sprint, for which program. `git blame` gives you author and date. Serro gives you program context and decision history on top.

**4. Contributor expertise graph.** A cross-signal view of who has worked on what — joined across GitHub, Slack, and docs. Not just who committed, but who's been in the threads and written the specs.

**5. Voice-driven live updates.** Program owners can update program memory via voice in real time. The update propagates immediately to the memory that downstream agents are reading.

**6. Collective agent governance.** Every AI agent in the org reads the same live program memory. They're not operating with isolated per-session context — they share a governed state.

**7. Large-scale org analyses.** On-demand rollups: what has engineering actually worked on this quarter, by program? Where has R&D investment gone?

The real moat across all of these isn't any single feature. It's the accumulating corpus — the data store that gets richer every week. That's what's structurally hard to replicate.

---

## The First Attempt and Why It Hit a Wall

The obvious starting point: Claude Code ships with MCP integrations for GitHub, Slack, Google Drive, and Google Calendar. Wire them all up, schedule a daily Claude agent to pull signals from each source, classify them into programs, and write them to markdown files in a shared repo.

This is a reasonable first attempt. It's also structurally wrong in one important way.

**MCP is pull-only.** Claude calls the MCP server. The MCP server never calls Claude. There are no event subscriptions, no webhooks, no push model anywhere in the MCP protocol. So what looks like "continuous signal ingestion" is actually polling — Claude wakes up on a schedule and asks "what happened since I last checked?"

With a daily schedule, that means up to 24 hours of signal lag. A critical decision made in Slack at 3pm is invisible to every AI agent in your org until 8am the next day. That's not living memory. That's a nightly batch job.

---

## The Harder Question: Do You Even Need Ingestion?

Once we identified the polling problem, the right move wasn't to immediately design a better ingestion pipeline. It was to question the premise.

GitHub already has all your PRs — live, accurate, queryable. Slack has all your messages. Drive has all your docs. If Claude can call those MCPs on demand, why copy anything?

For simple per-source queries, the answer is: you don't need ingestion. "What PRs merged this week?" — call the GitHub MCP directly. Live data, zero infrastructure, no staleness possible.

Ingestion earns its keep in one specific case: **cross-source joins**.

The MCP integrations are siloed. GitHub doesn't know about Slack. Slack doesn't know what a "program" is. None of them share a taxonomy. The queries that matter most — "what's blocking enterprise-readiness right now?", "who should review this PR?", "has this architectural decision been made before?" — require answers that span all three sources simultaneously.

| Query | GitHub | Slack | Drive | Why you need all three |
|---|---|---|---|---|
| What's blocking enterprise-readiness? | Open PRs, failing CI | Recent blocker threads | Charter scope, action items | Blockers exist in all three |
| Who should review this PR? | Commit history | Recent discussion | Doc authorship | GitHub alone misses the spec author |
| Has this decision been made before? | PR descriptions | Past threads | Design docs | Need all three to say "no" confidently |
| What shipped in Q2 for this program? | Merged PRs | Announcements | Published specs | Quarterly rollup is inherently multi-source |

So ingestion isn't about copying data. It's about adding a program classification layer that the source systems don't have. Something has to do the cross-source join — the question is whether it happens ahead of time (ingestion) or at query time (live synthesis).

---

## The Fork: Two Families of Solutions

This question splits the architecture into two families.

### Family B — Human-Maintained Source Mapping

Each program declares its sources in a config file. Which Slack channels, which GitHub repos, which Drive folders, which recurring meetings. At query time, Claude reads the config and fires targeted MCP calls against only those declared sources.

```yaml
# programs_to_sources_mapping.yaml
enterprise-readiness:
  slack_channels: ["#enterprise-readiness", "#sales-eng", "#security"]
  github_repos: [auth-service, billing-service, admin-portal]
  drive_folders: ["Programs/Enterprise Readiness"]
  recurring_meetings: ["Weekly Enterprise Sync"]
```

This is operationally elegant. Zero infrastructure. Always-live data. Cross-source joins work because the mapping tells Claude exactly where to look.

The accepted tradeoff: humans have to maintain the mapping. A new Slack channel spins up for a program and nobody adds it — that channel is silently excluded from every query, with no warning. For a disciplined team that explicitly owns this config as a lightweight operational responsibility, Family B is the right choice. For a team that wants automatic coverage with no maintenance, it isn't.

### Family C — Auto-Ingestion

Signals are discovered and written to memory automatically, without a human maintaining source declarations. Three implementation paths:

**C1 — Webhook server.** An always-on server receives events from GitHub, Slack, and Drive webhooks in real time and triggers `claude ingest` on each. Seconds of latency. Highest fidelity to Serro's architecture. Most complex to operate.

**C2 — Shared git repo + scheduled Claude agent.** A git repo holds program memory as markdown files. A scheduled Claude agent runs hourly, polls the MCPs for recent signals, classifies them into programs, and commits new entries. Any agent anywhere does `git pull` to read the latest state.

The git layer is the interesting part here: every memory update is a commit, giving you a versioned history of how organizational memory evolved. Human-readable, diffable, auditable, no database.

The honest tradeoff: up to 60 minutes of signal lag. Probably acceptable for most program queries.

**C3 — GitHub Actions + stateless serverless forwarder.** GitHub Actions handles the compute — no server to operate. GitHub events trigger Actions natively. A 10-line Cloudflare Worker (free tier) forwards Slack and Drive events to GitHub's `repository_dispatch` API, which fires the same Action. Near real-time (1–2 minutes). The only code you write and own is the Worker.

---

## The Recommendation

Start with **C2**. It works today, requires zero new infrastructure, and the git-as-memory-store pattern carries forward cleanly if you later migrate to C3. The shared git repo with hourly Claude ingestion is the right starting point for any team that wants automatic coverage without committing to operating infrastructure.

Migrate to **C3** when the hourly lag becomes a real problem. The Cloudflare Worker is the only meaningful addition.

Only consider **C1** if you need sub-10-second latency and are willing to own a server.

```
C2 now → C3 when lag matters → C1 only if necessary
```

---

## What's Next

We have the architecture mapped. We know what we're building: a shared git repo, a `programs_to_sources_mapping.yaml` to scope queries, a scheduled Claude agent for hourly ingestion, and MCP integrations for GitHub, Slack, Google Drive, and Google Meet.

The next checkpoint is the actual build — standing up C2, running it against a real program, and measuring how it performs against Serro's capabilities one by one.

Everything from this session is open source. The capability map, the approaches log, the ingestion architecture options — all in the repo. Link below.

---

*Part of an ongoing series: building an open-source Serro replica using only Claude Code.*
