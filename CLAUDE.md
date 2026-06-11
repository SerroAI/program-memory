# CLAUDE.md - Agent Navigation Instructions

## What this repo is
An open-source reference implementation for building a program memory layer using Claude Code and native MCP integrations. Covers Levels 1–3 (pull, map, loop). The proactive layer, widget layer, and graph/semantic index are out of scope.

## How to navigate (read in this order)
1. `README.md` — what this repo builds, scope boundary, quick start
2. `verdict.md` — pick your level (1–3) and your Family (A/B/C)
3. `memory_layer_decision_chart.md` — decision tree if you want the full decision rationale
4. `family_a/` — Level 1: full context pull (micro-orgs, zero config)
5. `family_b/` — Level 2: manual source mapping via yaml
6. `family_c/` — Level 3: auto-ingestion; start with `c4_loop.md`
7. `templates/` — copy-paste starting points

## Key files for agents
- `verdict.md` — single source of truth for level selection
- `templates/program_mappings.yaml` — the config file all approaches depend on
- `family_a/instructions.md` — Level 1 setup
- `family_b/instructions.md` — Level 2 setup
- `family_c/c4_loop.md` — Level 3 recommended starting point
- `family_c/instructions.md` — Level 3 option comparison (C-4 / C-2 / C-3 / C-1)

## Scope
**In scope (this repo):**
- Level 1 — Full Context Pull (Family A)
- Level 2 — Manual Source Mapping (Family B)
- Level 3 — Auto-Ingestion Loop (Family C)

**Out of scope:**
- Level 4 — Semantic/graph index (see `family_c/cocoindex_upgrade.md` and `family_c/laserdata_upgrade.md` for pointers)
- Proactive monitoring layer
- Action item follow-through
- Widget layer

## What this is NOT
- A finished product
- A codebase to run
- A substitute for Serro's managed platform (enterprise controls, entity resolution, temporal intelligence, managed infrastructure)
