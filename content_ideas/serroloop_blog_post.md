# Serroloop: What Happens When Program Engineering Meets Anthropic's Loop

> Blog post. Jake's voice. Audience: engineering leaders, senior ICs, TPMs who are starting to think about agentic tooling. Not a tutorial — a vision piece.

---

Every engineering org has programs. Most of them don't know it.

A program isn't a project. A program is a named, sustained, cross-functional commitment to an outcome that nobody finishes in a sprint. Enterprise readiness. Platform reliability. The mobile launch that's been "almost done" for two quarters. These things run across repos, across teams, across Slack channels, across meetings — and someone has to hold the picture.

That someone is doing program engineering. Usually without the title, usually on top of their actual job, and usually by holding an unsustainable amount of context in their head.

This is the problem Serro was built to solve. And it's the problem I've been thinking about since Anthropic introduced the concept of a loop.

---

## What a Loop Is

Anthropic's loop is simple: an autonomous agent that runs on a recurring interval. It wakes up. It reads state. It decides what to do. It acts. It goes back to sleep.

That's it. No user sitting there asking questions. No session starting from scratch each time. The agent maintains context across runs, accumulates a picture of what's happening, and surfaces what matters without being asked.

The loop is the primitive. The question is what you point it at.

---

## What Program Engineering Actually Requires

Program engineering has one core job: keeping visibility alive across a distributed, always-changing initiative.

That means:
- Knowing what shipped and what didn't
- Knowing what decisions were made and which ones are still open
- Knowing when something is blocked before the person who's blocked tells you
- Knowing when scope has drifted from what was agreed
- Knowing who's carrying the load and who's gone quiet

A human TPM does this by reading everything, attending every relevant meeting, and synthesizing it into a picture that they share with whoever needs it. The cost is high. The context lives in one person's head. When they're out, the picture goes dark.

---

## Serroloop

Here's what you get when you combine program memory with a loop.

The memory layer is the shared repo we've been building — a `program-memory` repository with a `programs_to_sources_mapping.yaml` that declares which GitHub repos, Slack channels, Drive folders, and meeting series belong to each program. Claude reads it before answering any program question. A digest gets written every night.

The loop layer runs on top of it. Every few hours, it wakes up and does what a TPM does:

1. Reads the current digest for each active program
2. Pulls live signals from declared sources — new PRs, merged commits, Slack threads since last run
3. Compares current state against the last known state
4. Asks: what changed? what's stalled? what looks like a blocker?
5. Writes a brief update to the program memory repo
6. Posts a summary to the right Slack channel if anything is worth flagging

No human initiates this. It just runs.

---

## What It Catches

The things a loop catches are the things that fall through the cracks in every eng org I've worked in:

**The PR that's been open for six days.** It was reviewed, comments were left, nobody responded. The loop sees it. It surfaces it. Nobody had to go looking.

**The Slack decision that never landed anywhere.** A thread in #platform reached a conclusion three days ago. That conclusion isn't in any doc, isn't in any PR description, isn't in any ticket. The loop reads the thread, classifies it as a decision for the platform-reliability program, and writes it to memory.

**Scope drift.** The mobile-launch program's mapping declares `org/mobile-api` as a source. The loop notices five of the last eight PRs touching `org/backend` mention "mobile" in their descriptions. That repo isn't in the mapping. Flag it. Maybe it should be.

**The program that's gone quiet.** No commits in the declared repos for four days. No Slack messages in the declared channels. Either the program is done and nobody closed it, or something's wrong. Either way, worth a note.

---

## How to Build It

The memory layer is documented in full at [the open-source repo](https://github.com/SerroAI/program-memory). Set that up first — Family B or Option C-2 are the right starting points for most teams.

The loop is a Claude Code invocation on a cron or workflow trigger, pointed at the program-memory repo:

```bash
# Runs every 4 hours
claude --project ~/program-memory "
You are a program engineering agent. Your job is to keep program visibility alive.

Read programs_to_sources_mapping.yaml and the most recent digest in digests/.

For each active program:
1. Pull live signals from declared sources since the last digest
2. Compare against the digest — what's new, what's stalled, what's missing?
3. Flag anything that looks like a blocker, an open decision, or scope drift
4. Write a brief update to digests/loop-YYYY-MM-DD-HH.md
5. If anything is flag-worthy, post a one-paragraph summary to the program's Slack channel

Be specific. Cite sources. Don't flag things that aren't actually notable.
"
```

That's the skeleton. The sophistication lives in what you tell the loop to look for and how you tune what counts as notable.

---

## What This Isn't

This isn't Serro. Serro has three years of accumulated corpus, temporal code symbol understanding, and a contributor expertise graph that this doesn't have. The loop I just described is operating on recent history — what happened in the last 24-48 hours. It can't answer "how has this program drifted over two quarters" without a longer memory store.

But for keeping visibility alive week to week? For catching the things that fall through the cracks because everyone's heads-down? For making program engineering something the whole team has instead of one person hoarding?

That's in reach. Today. With a cron job and a markdown file.

---

## The Bigger Idea

Program engineering is a discipline that most orgs practice badly, or not at all, because the cost of doing it well is too high. One person can hold the picture for one program. Two programs is a stretch. Three is a full-time job.

Loops change the math.

When visibility maintenance is autonomous — when the agent is doing the reading and synthesizing and flagging on a schedule — program engineering stops being a headcount question. It becomes infrastructure. The loop is the TPM that never sleeps, never loses context between runs, and doesn't have opinions about which programs matter more.

Serroloop isn't a product name. It's a pattern: shared program memory, plus an autonomous loop that keeps it alive. Build it once and every program in your org gets a TPM.

---

*The memory layer is open source. Start there: [link]. The loop is one cron job on top of it.*
