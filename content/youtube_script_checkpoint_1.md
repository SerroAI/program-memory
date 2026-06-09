# YouTube Script - Checkpoint 1
## "I'm Building a Serro Killer with Claude Code"

> Status: rough script / talking points for first segment. Not final. Updated as the project evolves.

---

### HOOK (0:00–0:30)

[Screen: Serro website or demo]

"There's a startup called Serro. It's positioning itself as living organizational memory for AI-native engineering teams. The idea is: every signal from your GitHub, your Slack, your meetings, your docs - automatically organized by program, always up to date, always available to any AI agent in your org.

It's a good idea. Maybe a great one.

I'm going to build it. Open source. Using only Claude Code and the integrations it ships with. And I'm going to show you exactly what breaks, what works, and what the real architectural decisions are.

This is checkpoint one."

---

### SECTION 1 - What is Serro Actually? (0:30–2:30)

[Screen: serro_capabilities.md]

"Before you can replicate something, you have to understand it precisely. Not the marketing - the mechanics.

Serro has seven core capabilities. Let me walk through them fast.

One: **Program-structured memory.** Everything in Serro is organized by program - not by date, not by repo, not by person. A program is a named, sustained, cross-functional commitment to an outcome. Think Enterprise Readiness. Platform Migration. Discoverability. Each one is a bucket that all related signals flow into.

Two: **Continuous signal ingestion.** GitHub commits, PRs, Slack threads, meeting transcripts, Google Docs - all automatically classified into the right program without anyone manually filing anything.

Three: **Temporal code symbol understanding.** Serro knows that `paymentService.processRefund` was touched by Alice in week 3 of the enterprise-readiness push, and why. Not just git blame - program context layered on top.

Four: **Contributor expertise graph.** Cross-referenced across GitHub, Slack, and docs. Who actually knows the billing system - not just who committed to it, but who's been in the threads and written the specs.

Five: **Voice-driven live updates.** A program owner can say 'we're cutting mobile from scope this quarter' and it propagates to memory immediately.

Six: **Collective agent governance.** Every AI agent in the org reads the same live program memory. They're not operating with isolated per-session context. They share a brain.

Seven: **Large-scale org analyses.** On demand: what has engineering actually worked on this quarter, by program? Where has R&D investment gone?

[beat]

The real moat isn't any single one of these. It's the accumulating corpus - the data store that gets richer every week. That's what's hard to replicate. That's what we need to crack."

---

### SECTION 2 - The First Attempt and Why It Hit a Wall (2:30–5:00)

[Screen: claude_generated_instructions.md]

"My first instinct was obvious. Claude Code has MCP integrations - GitHub, Slack, Google Drive, Google Calendar. Wire them all up. Schedule a Claude agent to run daily, pull signals from each source, classify them into programs, write them to markdown files. Done.

[beat]

Except MCP doesn't work that way.

MCP is pull-only. Claude calls the MCP server. The MCP server never calls Claude. There are no webhooks, no event subscriptions, no push. So what I described isn't continuous ingestion - it's polling. Claude wakes up once a day and asks 'what happened?' That's a 24-hour lag. A critical decision gets made in Slack at 3pm and every AI agent in your org is blind to it until 8am the next morning.

That's not living memory. That's a daily digest.

[pause]

But then I asked a harder question. Do we actually need to ingest at all?"

---

### SECTION 3 - The Real Question: Do You Even Need Ingestion? (5:00–8:00)

[Screen: approaches.md - The Deeper Question section]

"Here's the thing. GitHub already has all your PRs. Slack already has all your messages. Drive already has all your docs. They're live, they're accurate, they're queryable. If Claude can call those MCPs on demand - why copy anything?

For simple per-source queries, you don't need ingestion. 'What PRs merged this week?' - just call GitHub MCP. Live, accurate, zero infrastructure.

The place ingestion earns its keep is **cross-source joins**. GitHub doesn't know about Slack. Slack doesn't know what a program is. None of them share a taxonomy. If you need to answer 'what's blocking enterprise-readiness right now across GitHub, Slack, and Drive?' - something has to do that join. The question is when: ahead of time through ingestion, or at query time through live synthesis.

[visual: table of cross-source use cases]

Real queries that need the join: 'who should review this PR' - needs GitHub history plus Slack participation plus doc authorship. 'Has this decision been made before' - needs to search all three sources to say no confidently. 'What did we actually ship in Q2 for this program' - inherently multi-source.

So ingestion isn't about copying data. It's about adding a classification layer that the source systems don't have."

---

### SECTION 4 - The Fork: Family B vs Family C (8:00–11:00)

[Screen: approaches.md - The Fork in the Road]

"Once you accept that some form of cross-source indexing is needed, the architecture forks into two families.

**Family B** is human-maintained. Each program declares its sources in a config file - which Slack channels, which repos, which Drive folders, which recurring meetings. When a query comes in, Claude reads the config and fires targeted MCP calls against only those sources. Live data. No ingestion. No infrastructure.

[show programs_to_sources_mapping.yaml]

This is elegant. Operationally near-zero. And it works - as long as you explicitly accept one tradeoff: humans have to keep the config updated. A new channel spins up for a program and nobody adds it to the mapping - that channel is invisibly excluded from every query. No warning. No error. Just a silent gap.

That might be fine for a disciplined team. It's not fine if you want the coverage to be automatic.

**Family C** is auto-discovered. Signals are ingested without human config maintenance. Three ways to do it.

C1: a webhook server - always-on, real-time, most complex.
C2: a shared git repo with scheduled polling - hourly lag, zero infrastructure, fully Claude-native.
C3: GitHub Actions plus a 10-line Cloudflare Worker - near real-time, no server, best balance.

[beat]

My recommendation: **start with C2**. A shared git repo where a scheduled Claude agent writes signal entries every hour. The git layer gives you versioned memory history for free. Any agent on any machine pulls the repo and reads the same state. When the hourly lag becomes a real problem - migrate to C3.

C2 today. C3 when you need it. C1 only if you need sub-10-second latency and are willing to run infrastructure."

---

### CLOSE / NEXT STEPS (11:00–12:00)

"So that's checkpoint one. We've mapped the seven capabilities. We've hit the first architectural wall - MCP polling isn't continuous ingestion. We've worked out the real question - cross-source joins - and we've landed on two viable families of solutions with a recommended path.

What we haven't built yet: the actual implementation. That's next. We're going to build the C2 version - shared git repo, scheduled Claude agent, MCP integrations - and measure it against Serro's capabilities one by one.

Every file from this session is open source. Link in the description. If you want to follow along or contribute - the repo is there.

See you at checkpoint two."

---

### VISUAL / EDITING NOTES
- Show terminal / Claude Code sessions when discussing MCP calls
- Show the .md files being written and pushed in real time if possible
- `approaches.md` is the best screen to show during the fork section
- `family_c_ingestion_approaches.md` architecture diagrams can be animated as slides
- Keep code snippets on screen long enough to read - don't flash them
