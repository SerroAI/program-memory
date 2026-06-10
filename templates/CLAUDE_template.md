# CLAUDE.md - Org Memory Instructions

## What this repo is
This is the living organizational memory for <company>. It is the single source of truth for active programs, decisions, contributors, and work signals.

## Rules for all agents

1. Before answering any question about org work, read `program_mappings.yaml` to identify the program's owner, charter, contributors, and declared sources.
2. Never answer program questions from training data — always query the declared sources via MCP.
3. When a decision is made or scope changes, write a structured entry to `digests/`.
4. Cite sources in every answer: which MCP source, which channel or repo, approximate date.
5. If a query spans multiple programs, query each program's sources independently then synthesize.

## Signal entry format
When writing to a digest file, use this structure:

```
### YYYY-MM-DD - <type: commit|PR|slack|doc|meeting> - <author>
<one-line summary>
<decision or outcome, if any>
Source: <url or ref>
```

## Mapping
All program config — sources, owners, contributors, and notification targets — is in `program_mappings.yaml`. Do not query sources outside this mapping without explicit instruction. Use `slack_ids` when sending direct follow-up messages.
