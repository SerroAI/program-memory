# CLAUDE.md - Org Memory Instructions

## What this repo is
This is the living organizational memory for <company>. It is the single source of truth for active programs, decisions, contributors, and work signals.

## Rules for all agents

1. Before answering any question about org work, read `program_mappings.yaml` to identify the program's owner, charter, and declared sources.
2. Read `people_mappings.yaml` to identify who has context and who to attribute signals to.
3. Never answer program questions from training data — always query the declared sources via MCP.
4. When a decision is made or scope changes, write a structured entry to `digests/`.
5. Cite sources in every answer: which MCP source, which channel or repo, approximate date.
6. If a query spans multiple programs, query each program's sources independently then synthesize.

## Signal entry format
When writing to a digest file, use this structure:

```
### YYYY-MM-DD - <type: commit|PR|slack|doc|meeting> - <author>
<one-line summary>
<decision or outcome, if any>
Source: <url or ref>
```

## Source mapping
All program sources are declared in `program_mappings.yaml`. Do not query sources outside this mapping without explicit instruction.

## People mapping
Contributor context and notification targets are in `people_mappings.yaml`. Use this when attributing signals, selecting reviewers, or following up on action items.
