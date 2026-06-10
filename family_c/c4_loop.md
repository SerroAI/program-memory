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

## What goes in your serro-diy repo

```
serro-diy/
  CLAUDE.md                          ← how Claude answers program questions
  LOOP.md                            ← ingestion instructions the loop runs each iteration
  programs.md                        ← active programs and owners
  programs_to_sources_mapping.yaml   ← which sources belong to each program
  digests/                           ← written by the loop, one file per run
```

`CLAUDE.md` and `programs_to_sources_mapping.yaml` are the same as Family B — see [B2](../family_b/instructions.md#b2--create-a-shared-serro-diy-repo) and [B3](../family_b/instructions.md#b3--build-the-mapping-file).

`LOOP.md` is what makes C-4 different. It contains the exact instructions Claude runs on every loop iteration. Copy this into your repo:

---

**`LOOP.md`** — copy this file into your `serro-diy` repo as-is, then edit the notes at the top to match your org:

```markdown
# Loop Instructions

You are the program memory agent for this org. Run these instructions on every iteration.

---

## Step 1 — Read the source mapping

Read `programs_to_sources_mapping.yaml` to get the list of active programs and their
declared sources (GitHub repos, Slack channels, Drive folders).

## Step 2 — Find the last run timestamp

Read the most recent file in `digests/` and note its `last_run` timestamp.
If no digest exists yet, treat `last_run` as 24 hours ago.

## Step 2b — Check MCP connectivity

Before pulling any signals, verify that each MCP server is reachable. For any server
that is unavailable, skip all sources it handles for every program and write
`[MCP unavailable — skipped]` in those sections of the digest. Do not attempt to call
a disconnected server.

## Step 3 — Pull signals for each program

For each program in the mapping, pull only what's new since `last_run`. Pass the
timestamp as a filter directly to each MCP tool call — do not pull full history and
filter after the fact.

**GitHub** — for each declared repo:
- Commits merged to main: pass `since` = `last_run` (ISO 8601, e.g. `2024-01-15T14:30:00Z`)
- PRs opened, merged, or closed: filter by `updated_at >= last_run` (ISO 8601)
- Issues opened or closed: pass `since` = `last_run` (ISO 8601)
Include: title, author, PR/issue number, one-line summary.

**Slack** — for each declared channel:
- Pass `oldest` = `last_run` converted to a Unix timestamp (integer seconds since epoch)
  Example: `2024-01-15T14:30:00Z` → `1705329000`
- Surface threads with decisions, blockers, or scope changes only — not every message.

**Drive** — for each declared folder:
- Query `modifiedTime > '[last_run]'` (RFC 3339 format; ISO 8601 is compatible)
  Example: `modifiedTime > '2024-01-15T14:30:00Z'`
- Note: doc title and what changed, if visible.

If a source is unreachable, write `[source unreachable — skipped]` under that section
rather than leaving it blank. Do not retry — move on and note it.

## Step 4 — Write the digest

Create a new file at `digests/YYYY-MM-DD-HH.md` with this structure for every program
in the mapping (include programs with no activity — do not skip them):

---
last_run: [ISO 8601 timestamp]

## [program-name]
**GitHub**: [what shipped or changed — titles, numbers, authors]
**Slack**: [key decisions, blockers, or threads worth noting]
**Drive**: [docs updated and what changed]
**Status**: [one sentence on where this program stands right now]
**Watch**: [open blockers or unresolved decisions — leave blank if none]

Write "No activity." under any section with nothing to report.

## Step 5 — Commit and push

git add digests/
git commit -m "digest: [YYYY-MM-DD HH:MM]"
git push

## Step 6 — Self-pace

Call ScheduleWakeup with a delay based on what you found:
- Significant activity across 2+ programs → `delaySeconds: 1800` (30 min)
- Activity in 1 program → `delaySeconds: 3600` (1 hour)
- No activity across all programs → `delaySeconds: 7200` (2 hours)
```

---

## How to run it

**Step 1 — Configure and verify connectors**

If you haven't already, connect GitHub, Slack, and Drive in Claude Code settings →
Connectors (or `claude mcp add`). These credentials persist in your user config and
are used by all Claude Code sessions, including the loop.

Then verify all three are live:

```bash
/mcp
```

All three (github, slack, gdrive) should show connected. The loop checks connectivity
on every iteration and skips any source whose MCP is unavailable rather than failing.

**Step 2 — Start the loop**

```bash
cd ~/serro-diy
claude
```

Then in the session:

```
/loop Read LOOP.md and follow those instructions on every iteration.
```

That's it. Claude reads `LOOP.md`, follows the steps, writes a digest, commits, and self-paces.

**Fixed interval (optional)**

```
/loop 1h Read LOOP.md and follow those instructions.
```

Remove the self-pace step from `LOOP.md` if using a fixed interval.

**Step 3 — Verify the first run**

```bash
git log --oneline -3
ls digests/
```

Every program in the mapping should have a block in the digest, even ones with no activity.

**Step 4 — Let it run**

Leave the session open. The loop wakes itself up and keeps memory current. If the session closes, restart with the same command — it picks up from the last digest timestamp.

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
