# Family B: Manual Source Mapping — Instructions

> For orgs that have outgrown full context pull. You declare which sources belong to each program in a yaml file. Claude queries only those sources at query time — no ingestion, no infrastructure.

---

## Steps

- [B1 — Connect your tools](#b1--connect-your-tools)
- [B2 — Name your programs](#b2--name-your-programs)
- [B3 — Build the mapping file](#b3--build-the-mapping-file)
- [B4 — Write your CLAUDE.md](#b4--write-your-claudemd)
- [B5 — Run the context window test](#b5--run-the-context-window-test)
- [B6 — Set up the mapping health check](#b6--set-up-the-mapping-health-check)
- [B7 — Assign a mapping owner](#b7--assign-a-mapping-owner)
- [B8 — Define your upgrade trigger](#b8--define-your-upgrade-trigger)

---

## B1 — Connect your tools

Same as Family A. Install MCP servers for all sources your programs touch.

```bash
claude mcp add github --transport http https://api.githubcopilot.com/mcp/
claude mcp add slack --transport http https://mcp.slack.com/
claude mcp add gdrive --transport http https://mcp.gdrive.com/
```

See [Family A — A1](../family_a/instructions.md#a1--connect-your-tools) for full setup details per source.

Run `/mcp` to verify all servers are connected before proceeding.

---

## B2 — Name your programs

Define each active program with a name, owner, and one-line charter. This becomes the index Claude uses to scope queries.

**Example: `programs.md`**

```markdown
# Active Programs

## auth-modernization
Owner: @sarah
Charter: Replace legacy session tokens with JWT across all services. Q3 target.

## platform-reliability  
Owner: @marcus
Charter: Reduce p99 latency below 200ms across all API endpoints.

## mobile-launch
Owner: @priya
Charter: Ship iOS and Android v1.0 by end of Q4.
```

Use lowercase-hyphenated names — these become the keys in your mapping file.

---

## B3 — Build the mapping file

Create `programs_to_sources_mapping.yaml` at your repo root. For each program, declare every source that contains signals relevant to it.

```yaml
# programs_to_sources_mapping.yaml
# One entry per program. Be explicit — unmapped sources are invisible.

auth-modernization:
  github:
    repos:
      - org/backend
      - org/auth-service
    label_filter: "auth"           # optional: only PRs/issues with this label
  slack:
    channels:
      - auth-migration
      - backend-eng
      - security
  drive:
    folders:
      - "Engineering/Auth RFCs"
      - "Engineering/Security Reviews"
  meetings:
    series:
      - "Auth Weekly Sync"         # match against transcript titles

platform-reliability:
  github:
    repos:
      - org/backend
      - org/infra
      - org/api-gateway
  slack:
    channels:
      - platform
      - incidents
      - on-call
  drive:
    folders:
      - "Engineering/Reliability"
      - "Engineering/Postmortems"
  meetings:
    series:
      - "Platform Reliability Review"

mobile-launch:
  github:
    repos:
      - org/ios
      - org/android
      - org/mobile-api
  slack:
    channels:
      - mobile
      - mobile-launch
      - design
  drive:
    folders:
      - "Mobile/Launch Plan"
      - "Mobile/Design Specs"
```

**Rules for a good mapping:**
- Every source that a program's work touches should be listed
- If a channel is relevant to two programs, list it under both
- Err on the side of over-mapping — you can narrow later
- Don't list sources just because they exist — only sources that contain signal for that program

---

## B4 — Write your CLAUDE.md

`CLAUDE.md` tells Claude to use the mapping file and query only the declared sources for the active program.

**Place at repo root: `CLAUDE.md`**

```markdown
# Program Memory — Claude Instructions

## Source mapping

All program sources are declared in `programs_to_sources_mapping.yaml`.

When answering a question about a program:
1. Read `programs_to_sources_mapping.yaml` to find the sources for that program
2. Query only those sources — do not pull from unmapped sources
3. For each source type, use the appropriate MCP tool:
   - GitHub: search commits, PRs, issues in the declared repos
   - Slack: search threads and messages in the declared channels
   - Drive: search documents in the declared folders
4. Synthesize across sources — cite the source of each key claim
5. If a source returns no results, say so — do not skip it silently

## Constraints

- Do not query sources outside the mapping for the active program
- Do not answer from memory alone — always pull fresh context
- If the question spans multiple programs, query each program's sources separately

## Programs

See `programs.md` for program names, owners, and charters.
```

---

## B5 — Run the context window test

Before committing to Family B, verify that querying all sources for your largest program fits in Claude's context window without degradation.

**Run this test for each program:**

1. Ask Claude a broad question that should draw from all sources:
   ```
   Give me a full status update on [program name]: recent decisions, open blockers,
   who's been active, and any scope changes in the last 30 days.
   ```

2. Measure:
   - **Total tokens returned** — check Claude's context usage
   - **Latency** — how long did it take end-to-end?
   - **Accuracy** — compare the answer against what you know to be true
   - **Truncation** — did Claude say it couldn't fit everything?

**Pass criteria:**
- No truncation warnings
- Latency under 60 seconds
- Answer matches ground truth on at least 80% of verifiable claims

**If the test fails:** see workarounds in [`family_b/overview.md`](overview.md) and consider whether Family C is the right path.

---

## B6 — Set up the mapping health check

Family B fails silently when the mapping is stale. A health check catches the most common failure mode: a channel gets renamed, a repo gets archived, or a folder gets moved.

**Option A — Shell script + cron** ⚙️

```bash
#!/bin/bash
# check_mapping_health.sh
# Reads programs_to_sources_mapping.yaml and verifies each declared source exists

# Requires: yq (yaml parser), GitHub CLI (gh), Slack CLI or API token

MAPPING="programs_to_sources_mapping.yaml"
ERRORS=0

# Check GitHub repos
for repo in $(yq '.*.github.repos[]' $MAPPING 2>/dev/null); do
  if ! gh repo view "$repo" &>/dev/null; then
    echo "ERROR: GitHub repo not found: $repo"
    ERRORS=$((ERRORS + 1))
  fi
done

if [ $ERRORS -gt 0 ]; then
  echo "$ERRORS source(s) missing from mapping. Update programs_to_sources_mapping.yaml."
  exit 1
fi

echo "Mapping health check passed."
```

Schedule weekly:
```bash
# crontab -e
0 9 * * 1 /path/to/check_mapping_health.sh | mail -s "Mapping health check" owner@yourorg.com
```

**Option B — GitHub Actions scheduled workflow** ⚙️

```yaml
# .github/workflows/mapping-health.yml
name: Mapping Health Check
on:
  schedule:
    - cron: '0 9 * * 1'  # every Monday 9am UTC
  workflow_dispatch:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check mapping sources
        run: ./scripts/check_mapping_health.sh
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Option C — Claude scheduled agent** ⚙️

Ask Claude once a week:
```
Read programs_to_sources_mapping.yaml. For each declared GitHub repo, Slack channel,
and Drive folder, verify it still exists and is accessible. Report any that are missing
or inaccessible. Post the result to #program-ops.
```

Schedule this as a `/schedule` command or cron-triggered Claude invocation.

---

## B7 — Assign a mapping owner

A mapping without an owner drifts. Assign one person — not a team — who is responsible for keeping `programs_to_sources_mapping.yaml` current.

**What the mapping owner does:**
- Updates the mapping when a new channel, repo, or folder is created for a program
- Runs (or reviews) the health check output each week
- Owns the decision of which sources belong to which program
- Reviews the full mapping once per quarter against active program charters

Add the owner's name to the mapping file:

```yaml
# programs_to_sources_mapping.yaml
# Mapping owner: @sarah — update her when sources change

auth-modernization:
  ...
```

---

## B8 — Define your upgrade trigger

Family B has known limitations. Define now what signal tells you to move to Family C.

| Signal | Threshold |
|---|---|
| Context window | Queries for any program regularly exceed context limits or return truncated answers |
| Keyword search gaps | Questions about "how has X evolved" or "what does Y mean now" can't be answered accurately |
| Latency | Cross-source queries take more than 60 seconds |
| Query volume | More than 10 queries per hour across the org — rate limit risk becomes real |
| Coverage requirements | Leadership requires zero-gap coverage — silent gaps are no longer acceptable |

**When you hit the trigger:** see [`family_c/instructions.md`](../family_c/instructions.md).
