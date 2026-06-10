# CLAUDE.md - Agent Navigation Instructions

## What this repo is
A research and implementation guide for building an open-source Serro replica using Claude Code and native MCP integrations. Not a product - a documented journey from capability mapping through architectural decisions to working implementations.

## Repo structure (read in this order)
1. `goal.md` - the mission and personal motivation behind this project
2. `research/serro_capabilities.md` - the 7 Serro capabilities we're replicating, with measurement rubrics
3. `comparative_analysis.md` - what was evaluated and why, including dead ends and the key architectural fork
4. `key_decisions.md` - the 13 decision points with rationale
5. `decision_chart.md` - mermaid decision tree for picking an architecture
6. `family_a/` - Full Context Pull: no config, pull all sources at query time (micro-orgs)
7. `family_b/` - Manual Source Mapping: declare sources per program in yaml
8. `family_c/` - Auto-Ingestion: C1 (webhook server), C2 (git+cron), C3 (GitHub Actions)
9. `templates/` - copy-paste starting points: mapping yaml, charter, CLAUDE.md
10. `content/` - YouTube script and blog post drafts (not part of the tool)

## Key files for agents
- `comparative_analysis.md` - reasoning behind all architectural decisions
- `family_a/instructions.md` - full context pull setup (micro-orgs)
- `family_b/overview.md` - human-maintained approach (simpler, requires compliance)
- `family_b/instructions.md` - source mapping step-by-step
- `family_c/overview.md` - auto-ingestion approach (three options)
- `family_c/c2_git_cron.md` - recommended starting point for Family C
- `templates/program_mappings.yaml` - the core config file all approaches depend on

## What this is NOT
- A finished product
- A codebase to run
- A substitute for Serro's proprietary data corpus

## Current status
Checkpoint 1 complete: capabilities mapped, approaches evaluated, implementation instructions drafted. Not yet validated against a real org.
