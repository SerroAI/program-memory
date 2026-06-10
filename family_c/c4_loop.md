# Option C-4 - Claude Code Loop

**Latency:** Self-paced (15 min–2 hours) | **Complexity:** Low | **Infrastructure:** Persistent Claude Code process

## What it is

A Claude Code loop IS the ingestion pipeline. Instead of an external cron job invoking Claude as a subprocess, Claude runs as a persistent process and manages its own schedule. It wakes up, reads the mapping, pulls signals from all declared sources, writes to the memory store, commits, and decides when to run again — based on what it actually found.

No bash wrapper. No cron. No external scheduler. The loop primitive is the ingestion mechanism.

## Architecture

```
claude /loop "update program memory"
        ↓
Claude wakes up, reads programs_to_sources_mapping.yaml
        ↓
For each program:
  - GitHub MCP: commits, PRs, issues since last run
  - Slack MCP: messages in declared channels since last run
  - Drive MCP: docs modified in declared folders since last run
        ↓
Classifies signals into programs
Writes structured entries to digests/YYYY-MM-DD-HH.md
Updates .last_run timestamp
        ↓
git commit + git push
        ↓
Self-pace: found significant activity → sleep 30 min
           quiet across all programs → sleep 2 hours
        ↓
Wakes up and repeats
```

## How to run it

**Step 1 — Make sure MCPs are connected**

```bash
# Verify all MCP servers are live before starting the loop
/mcp
```

All three (github, slack, gdrive) should show connected. The loop will fail silently on any program whose sources use a disconnected MCP.

**Step 2 — Start the loop from inside the program-memory repo**

```bash
cd ~/program-memory
claude
```

Then in the session, paste the full loop prompt:

```
/loop You are the program memory agent for this org. On each iteration:

1. Read programs_to_sources_mapping.yaml to get the list of programs and their
   declared sources (repos, Slack channels, Drive folders).

2. Read the most recent file in digests/ to find the last_run timestamp.
   If no digest exists yet, treat last_run as 24 hours ago.

3. For each program in the mapping:
   a. GitHub: search for commits, merged PRs, and opened/closed issues in the
      declared repos since last_run. Include PR titles, authors, and numbers.
   b. Slack: search for messages and threads in the declared channels since
      last_run. Surface decisions, blockers, or notable discussions — not
      every message.
   c. Drive: list documents created or modified in the declared folders since
      last_run. Note the doc title and what changed if visible.

4. Write a new digest file at digests/YYYY-MM-DD-HH.md with this structure
   for each program:

   ## [program-name]
   last_run: [ISO timestamp]
   **GitHub**: [what shipped or changed — PR titles, numbers, authors]
   **Slack**: [key decisions, blockers, or threads worth noting]
   **Drive**: [docs updated and what changed]
   **Status**: [one sentence on where this program stands right now]
   **Watch**: [anything that looks like a blocker or open decision]

   Write "No activity." under any section with nothing to report.
   Write the full program block even if there was no activity — omitting it
   means the next iteration can't tell the difference between quiet and missing.

5. Commit the new digest file and push:
   git add digests/
   git commit -m "digest: [timestamp]"
   git push

6. Self-pace your next run:
   - Significant activity across 2+ programs → run again in 30 minutes
   - Moderate activity (1 program active) → run again in 1 hour
   - No activity across all programs → run again in 2 hours
```

**Step 3 — Verify the first run**

After the first iteration completes, check that a new file appeared in `digests/` and was committed:

```bash
git log --oneline -3
ls digests/
```

The digest should have an entry for every program in the mapping, even ones with no activity.

**Step 4 — Let it run**

Leave the Claude Code session open. The loop will wake itself up on the self-paced schedule and keep the memory current. If you close the session, the loop stops — restart it with the same `/loop` command and it will pick up from the last digest timestamp.

**Fixed interval (optional)**

If you'd rather have predictable run times instead of self-pacing:

```
/loop 1h [same prompt as above]
```

Replace `1h` with `30m`, `2h`, etc. Fixed intervals ignore the self-pace instruction — remove it from the prompt if you use this form.

## Key distinction from Option C-2

| | Option C-2 | Option C-4 |
|---|---|---|
| What triggers runs | External cron | Claude itself |
| Interval | Fixed (every N minutes) | Adaptive — Claude decides based on activity |
| Context across runs | None — each run starts cold | Persistent within the session |
| Setup | Bash script + cron config | One `/loop` command |
| Runs headlessly? | Yes — no active session needed | Requires a running Claude Code process |
| Cloud-ready today? | Yes (GitHub Actions, any server) | Needs a persistent host; cloud workflows coming |

## Pros
- No external scheduler — Claude owns the loop
- Self-pacing based on actual activity level
- Persistent context across iterations — Claude remembers what it saw last run
- Simplest setup of all Family C options: one command
- Natural migration path to Anthropic cloud workflows when they ship

## Cons
- Requires a persistent Claude Code process — not headless like C-2
- If the session dies, the loop stops (no auto-restart without a process manager)
- Self-paced intervals are less predictable than a fixed cron for SLA purposes
- Cloud workflow support not yet shipped — today this means a machine running Claude Code continuously

## Cloud workflows (coming)

Anthropic is shipping cloud-native workflow support for Claude Code. When it lands, Option C-4 becomes the cleanest production path: your loop runs in Anthropic's cloud on a schedule, no machine required, with full MCP connectivity to GitHub, Slack, and Drive. The `/loop` command you write today is the same one you'd deploy to cloud workflows — the trigger mechanism changes, the logic doesn't.

Until then, C-4 runs locally or on a persistent host. If you need fully headless operation today, use Option C-2 and migrate to C-4 when cloud workflows ship.

## When to use this over Option C-2

Use Option C-4 when:
- You want the simplest possible setup with no bash scripts or cron config
- Variable activity levels make a fixed interval wasteful (busy days = 30 min, quiet weekends = 2 hours)
- You're planning to migrate to Anthropic cloud workflows when they ship
- You're already running Claude Code interactively and want memory to update alongside your work

Stay with Option C-2 when:
- You need fully headless operation with no active session
- You want predictable, auditable run times (e.g. "memory updates at 6am daily")
- You're deploying on a server or in CI where Claude Code sessions don't persist

## Verdict

The most Claude-native ingestion option. Lowest setup cost of all Family C options. The main constraint is that it requires a persistent process — which matters less as Anthropic ships cloud-native workflow support. **Start with C-2 if you need headless operation today. Start with C-4 if you want the simplest setup and don't mind running Claude Code continuously.**
