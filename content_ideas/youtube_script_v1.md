# YouTube Script — v1
## "I Work at Serro. Here's How to Build It for Free."

> First-person. Jake's voice. Adversarial angle. Not a product walkthrough.

---

### HOOK (0:00–0:45)

[Screen: blank, just me talking to camera]

"I work at a company called Serro. My job is to build AI tools for engineering teams.

Serro's core product is organizational memory — it ingests everything your team produces: GitHub, Slack, meetings, docs — and makes it queryable by any AI agent in your org. Program-indexed, always live, always accurate.

It's a genuinely good idea. We've spent a long time building it.

And I think anyone can replicate it in a weekend.

That's what this video is. Not a teardown. Not a hit piece. Just an honest engineering answer to the question: what does it actually take to build this?"

---

### SECTION 1 — What Serro Is (and Isn't) (0:45–3:00)

[Screen: serro_capabilities.md]

"Let me be specific about what Serro does, because the marketing is vague and the mechanics matter.

Seven capabilities. I'll go fast.

**Program memory.** Everything organized by program — not by repo, not by person, not by date. A program is a named, sustained initiative. Enterprise Readiness. Platform Migration. Mobile Launch. All signals across GitHub, Slack, and docs flow into the right program automatically.

**Continuous ingestion.** New PR merged on auth-service? Classified. Decision made in #platform? Classified. Meeting transcript from the architecture review? Classified. No one files anything manually.

**Temporal code understanding.** Not just git blame. Serro knows this function was touched by Alice during the enterprise-readiness push, in the context of the SOC2 audit, because it connects the PR to the Slack thread to the meeting where the decision was made.

**Contributor graph.** Who actually knows the billing system — not just who committed to it, but who's been in the threads and written the specs. Real expertise, not just activity.

**Voice updates.** A PM can say 'we're cutting mobile from scope' and it lands in memory immediately. No ticket. No doc. Just said it.

**Shared agent context.** Every AI agent in your org reads the same memory. They're not each starting from scratch. They share a brain.

**Org-level analysis.** What did engineering actually work on this quarter? Where did R&D time go? By program, not by commit count.

[pause]

Here's the honest thing: capabilities one through six are buildable. They're hard, but buildable. The real moat is the corpus. Serro has been ingesting and indexing org signals since 2023. You can't replicate three years of accumulated context in a weekend. That gap is real and I'll come back to it.

But capabilities? Let's see."

---

### SECTION 2 — The Approach (3:00–5:30)

[Screen: terminal, Claude Code]

"The constraint I set for myself: Claude Code only. No custom backends. No vector databases I have to operate. No pipelines. Just Claude Code and the MCP integrations it ships with — GitHub, Slack, Google Drive.

The thesis is that Claude's context window is large enough, and its tool-use is good enough, that you can build most of this by writing instructions in markdown and wiring up the right integrations. No infrastructure.

First question I had to answer: do you even need to ingest data at all?

GitHub already has your PRs. Slack already has your messages. Drive already has your docs. They're live, accurate, and queryable. If Claude can call those MCPs on demand — why copy anything?

The answer is: you don't, until you need to join across them.

'What's blocking enterprise-readiness right now?' — that question touches GitHub, Slack, and Drive simultaneously. None of those systems know what 'enterprise-readiness' means. Something has to hold that mapping. The question is whether that something is a config file a human maintains, or an ingestion pipeline that figures it out automatically.

That fork is where the architecture lives."

---

### SECTION 3 — The Three Approaches (5:30–9:00)

[Screen: comparative_analysis.md, then family overviews]

"I landed on three families of solutions, ordered by complexity.

**Family A — Full Context Pull.**
For tiny orgs. No config. No mapping. Just point Claude at all your sources and let it pull everything at query time. Works when you have one repo, two Slack channels, and one program. Falls apart fast as you grow. But zero setup cost, zero maintenance. Start here if you're a 3-person team.

**Family B — Manual Source Mapping.**
You declare which sources belong to each program in a yaml file. Claude reads the mapping and queries only those sources. No ingestion, no infrastructure. This is the right approach for most small-to-mid-sized teams — disciplined, explicit, and fast to set up.

[show programs_to_sources_mapping.yaml]

The tradeoff is compliance. A new Slack channel spins up for a program and nobody updates the yaml — that channel is silently excluded from every query. No warning. For a team with a dedicated TPM or a manager who owns this operationally, it's fine. For a team that wants zero-maintenance coverage, it's not.

**Family C — Auto-Ingestion.**
Signals are discovered and stored automatically. Three sub-options.

Option C-2: a shared git repo, a scheduled Claude agent that polls every hour, writes summaries to markdown files, commits back. Zero new infrastructure. Git history gives you versioned memory for free. This is where I'd start.

Option C-3: GitHub Actions plus a 10-line Cloudflare Worker. Near real-time. No server to operate. Best balance for a team already on GitHub.

Option C-1: an always-on webhook server. Real-time. Full architectural parity with how Serro likely works internally. Only worth it if seconds matter.

[beat]

My recommendation: start with Family B if you have someone to own the mapping. Start with Option C-2 if you want it to run itself. Migrate to Option C-3 when the hourly lag becomes a real problem."

---

### SECTION 4 — What This Gets You (and What It Doesn't) (9:00–11:00)

[Screen: back to camera]

"Let me be straight about where this lands versus Serro.

Cross-source program queries? Covered. 'What's the status of auth modernization across GitHub, Slack, and docs?' — yes, this works.

Shared agent context? Covered. Point every Claude instance at the shared program-memory repo and they all read the same state.

Voice updates? Covered. Say it to Claude, it writes to the repo.

Org-level analysis? Covered for recent history. 'What shipped last week across all programs?' — yes.

Temporal code understanding? Partial. Claude can reason across recent PRs and Slack threads in context. But Serro's three-year corpus means it can answer 'when was this function last touched and why' going back to 2022. The open-source version starts from zero. That gap is real and it grows every week Serro is running.

Contributor expertise graph? Not covered yet. That's a harder problem — cross-referencing GitHub, Slack, and docs into a real expertise model. It's on the roadmap.

[pause]

The honest answer: this replicates about 70% of what Serro does for a small-to-mid-sized team, today, for free, in a day of setup. The 30% gap is the accumulated corpus and the deeper code intelligence layer. That gap matters more as your org grows and your history gets longer.

Whether 70% is good enough depends on your team size, your history depth, and how much you care about code-level understanding versus program-level coordination."

---

### CLOSE (11:00–12:00)

[Screen: repo]

"Everything I just described is open source. The repo has the complete implementation guides for all three families — step by step, copy-paste ready.

I'll be building this out incrementally and documenting what breaks. Checkpoint two is actually implementing Family B or Option C-2 against a real org and measuring classification accuracy, coverage, and latency against Serro's numbers.

If you want to follow along or contribute — link in the description.

And if you work at Serro and you're watching this — hi. I think this is good for the space. The market for AI-native org memory is real. Open-sourcing the basics raises all boats."

---

### NOTES FOR FILMING
- Open on camera, no screen — establish the insider angle before showing anything technical
- Section 2 onward: terminal and markdown files, not slides
- Don't rush the pause after "that gap is real" — let it land
- The close should feel direct, not apologetic about the conflict of interest
- Total runtime target: 11–12 minutes
