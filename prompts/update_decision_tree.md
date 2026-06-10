# Prompt: Update the Architecture Decision Tree

Paste this into a Claude session when you want to change `decision_chart.md`.

---

You are helping update the architecture decision tree for this open-source Serro replica repo.

## What the decision tree is

`decision_chart.md` contains a Mermaid `flowchart TD` that routes engineering orgs to the right level of live program memory. It maps to three canonical levels:

- **Level 1 (Family A)** — Pull everything at query time. No config, no maintenance.
- **Level 2 (Family B)** — Maintain `program_mappings.yaml`: programs, sources, people in one file.
- **Level 3 (Family C)** — A Claude loop automaintains digests. C-4 (Claude Code Loop) is the recommended starting point; C-2 (git + cron), C-3 (GitHub Actions), C-1 (webhook server) are operational alternatives.

The three-level framing in `verdict.md` and `README.md` is canonical. Any changes to the chart should stay consistent with it.

## Structure of the current chart

The flowchart has these sections — work through them in order when making changes:

1. **Context size gate** — Is your context under 20 sources? → Family A (Level 1)
2. **Long-horizon reasoning check** — Do you need semantic/historical queries? → Level 3
3. **Team discipline + silent gaps** — Will someone own the mapping file? → Family B (Level 2) viability
4. **Rate limits and context window test** — Final gate before committing to Level 2
5. **Family C option selection** — C-4 vs C-2 vs C-3 vs C-1 based on latency and infrastructure tolerance
6. **Memory validation gate** — Hard dependency before enabling proactive monitoring
7. **Proactive monitoring** — Serroloop pattern (loop reads memory, flags blockers, posts Slack digest)
8. **Action item follow-through** — Notify targets declared in `program_mappings.yaml`
9. **Widget layer** — Out of scope (checkpoint 1)

## Files to read before editing

1. `decision_chart.md` — the Mermaid source
2. `verdict.md` — the 3-level rationale; your changes must stay consistent with this
3. `key_decisions.md` — rationale behind each decision node (13 points)
4. `comparative_analysis.md` — what was tried and why it failed; don't re-open dead ends

## What to preserve

- All ⚠️ tradeoff/workaround nodes — they show costs and verdicts, not dead ends
- All 🧪 test/measurement nodes — they gate whether to proceed
- The memory validation gate (decision 10) — hard dependency for the proactive layer
- References to instruction files (`family_a/`, `family_b/`, `family_c/c4_loop.md`, etc.)
- C-4 as the recommended starting point for Level 3

## What you can change

Tell me what you want to change. Common update types:

- **Add a decision node** — describe the question, the condition, and where it routes
- **Change a verdict** — describe the new recommended path and why
- **Update Family C options** — e.g. changing the latency tiers, adding a new option
- **Reflect new level framing** — if the 3-level model changes, update routing nodes accordingly
- **Fix a routing error** — describe the current behavior and the correct behavior

After each change, I will update both the Mermaid source in `decision_chart.md` and the introductory summary table at the top of the file if the family descriptions change.
