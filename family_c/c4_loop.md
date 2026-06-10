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

## Caveats and workarounds

**The loop stops when your laptop sleeps or closes.**
This is the most common failure mode. Claude Code needs an active process — close the lid and the loop pauses until you reopen it. Memory goes stale overnight and on weekends.

*Workaround A:* Keep your laptop on and plugged in while the loop runs. Works for daytime coverage during the week. Not reliable for 24/7.

*Workaround B (recommended):* Run the loop on a dedicated always-on machine — a cheap VPS ($5/month), a spare Mac mini, or an old laptop that stays open. SSH in, start the loop, leave it. This is the cleanest path to reliable coverage until cloud workflows ship.

*Workaround C:* Combine with Option C-2. Let the loop cover daytime (when your laptop is on), and a cron on a server covers overnight and weekends. Both write to the same `digests/` folder — they don't conflict since each writes its own timestamped file.

*Coming:* Anthropic is shipping cloud-native workflow support for Claude Code. When it lands, you run the loop in Anthropic's cloud — no machine required, always on. The `/loop` prompt you write today is the same one you'd deploy. Until then, an always-on machine is the closest equivalent.

---

**The loop stops if the Claude Code session crashes or exits.**
No auto-restart out of the box.

*Workaround:* Wrap the Claude invocation in a restart loop in your shell:
```bash
# keep_loop_alive.sh
while true; do
  claude  # start session, paste /loop prompt manually
  echo "Session ended. Restarting in 10 seconds..."
  sleep 10
done
```
Or use `launchd` (macOS) or `pm2` to manage the process and restart on exit.

---

**Memory has gaps when the loop is offline.**
If the loop was down for 6 hours, those 6 hours aren't in any digest. The next run will catch up from the last timestamp — but anything that happened during the gap gets summarized in bulk, not incrementally.

*Workaround:* The catch-up run is usually fine. If a specific time window matters, note the gap in the digest manually or run Option C-2 as a fallback for that period.

---

**Network drops disconnect MCP servers mid-run.**
If Slack or GitHub MCP loses connection during an iteration, that source call fails. Claude will log the error and continue with the other sources — the digest will be incomplete for that program's failed source.

*Workaround:* The loop prompt instructs Claude to note when a source was unreachable rather than silently skipping it. Check the digest for `[source unreachable]` markers and re-run manually if coverage matters.

---

**Self-pacing is not a guarantee.**
Claude decides the interval based on what it found — but it's a judgment call, not a hard timer. Occasionally it will sleep longer than expected or shorter than needed.

*Workaround:* Use a fixed interval (`/loop 1h`) if you need predictable run times. Self-pacing is best for "keep it roughly current" use cases, not for SLA-bound pipelines.

---

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

**Best fit: individual program memory.** Option C-4 is a natural fit when one person (a TPM, EM, or program owner) is running memory for a program or set of programs they own. The loop runs from their machine, stays connected to their MCP credentials, and keeps their program's memory current.

**Organizational program memory — a future direction.** When memory spans a whole org and multiple people are contributing signals, a single loop on one person's machine starts to feel like a bottleneck. One direction worth exploring: each contributor runs their own loop, writing to a per-person `.md` file in the repo, and a reconciliation pass merges those into the shared program digest. Think of it like distributed commits that get resolved into a single canonical view. More on this in a future checkpoint.
