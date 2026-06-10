# Family B: Manual Source Mapping — Instructions

> For orgs that have outgrown full context pull. You declare which sources belong to each program in a yaml file. Claude queries only those sources at query time — no ingestion, no infrastructure.

---

## Steps

- [B1 — Connect your tools](#b1--connect-your-tools)
- [B2 — Create a shared serro-diy repo](#b2--create-a-shared-serro-diy-repo)
- [B3 — Build the mapping file](#b3--build-the-mapping-file)
- [B4 — Write your CLAUDE.md](#b4--write-your-claudemd)
- [B5 — Run the context window test](#b5--run-the-context-window-test)
- [B6 — How to keep it current](#b6--how-to-keep-it-current)
- [B7 — Set up the mapping health check](#b7--set-up-the-mapping-health-check)
- [B8 — Define your upgrade trigger](#b8--define-your-upgrade-trigger)

---

## B1 — Connect your tools

Same as Family A. Install MCP servers for all sources your programs touch.

```bash
claude mcp add github -- npx -y @modelcontextprotocol/server-github
claude mcp add slack -- npx -y @modelcontextprotocol/server-slack
claude mcp add gdrive -- npx -y @modelcontextprotocol/server-gdrive
```

See [Family A — A1](../family_a/instructions.md#a1--connect-your-tools) for environment variable setup, required OAuth scopes, and troubleshooting.

Run `/mcp` to verify all servers are connected before proceeding.

---

## B2 — Create a shared serro-diy repo

Same as Family A: create a dedicated repo that the whole team shares. This is where the mapping file lives, where Claude reads instructions, and where digest files get committed.

```bash
git clone https://github.com/your-org/serro-diy
cd serro-diy
```

Structure:

```
serro-diy/
  CLAUDE.md                 ← instructions for Claude
  program_mappings.yaml     ← sources, owner, and charter per program
  people_mappings.yaml      ← contributors, leads, and Slack IDs per program
  digests/                  ← daily digest files
  scripts/
    update_digest.sh        ← digest update script
    check_mapping_health.sh ← health check script
```

Why a separate repo:
- Everyone clones it — teammates, TPMs, managers — without needing access to product code
- The mapping files become a shared contract that any program owner can update via a PR
- Claude can commit digest updates back without touching your main repos

### Connecting teammates to this repo

`CLAUDE.md` only does anything when Claude is pointed at this repo. The mapping file is just yaml until Claude is instructed to read it. Every teammate connects once.

**Add it as a project in Claude Code**

From inside any other repo (your product code, wherever you normally work):

```
/project:add ~/path/to/serro-diy
```

Claude loads the shared `CLAUDE.md` alongside your current project. Program questions work from anywhere without switching directories.

For scripting and automation (e.g. the digest script), `cd` into the `serro-diy` repo before invoking `claude` so the right `CLAUDE.md` is loaded.

The `CLAUDE.md` is what converts a static mapping file into active query behavior. Without it loaded, Claude answers from its own knowledge — stale, uncited, and wrong for anything recent.

---

## B3 — Build program_mappings.yaml

Create `program_mappings.yaml` at your repo root. Each entry declares a program's owner, charter, and every source that contains signals for it.

```yaml
# program_mappings.yaml
# One entry per program. Be explicit — unmapped sources are invisible.

auth-modernization:
  owner: "@sarah"
  charter: "Replace legacy session tokens with JWT across all services. Q3 target."
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
  owner: "@marcus"
  charter: "Reduce p99 latency below 200ms across all API endpoints. Ongoing."
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
  owner: "@priya"
  charter: "Ship iOS and Android v1.0 by end of Q4."
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

## B3b — Build people_mappings.yaml

Create `people_mappings.yaml` at your repo root. Each entry declares who has context in a program, who to attribute signals to, and who to notify for action item follow-up.

```yaml
# people_mappings.yaml
# Who has context in each program. Update when roles change — not every sprint.

auth-modernization:
  owner: "@sarah"
  leads:
    - "@marcus"                    # tech lead
  contributors:
    - "@alex"
    - "@jordan"
  notify:                          # pinged when action items go stale
    - "@sarah"
    - "@marcus"
  slack_ids:                       # needed for DM follow-ups via Slack API
    "@sarah": "U04A1B2C3D4"        # right-click user in Slack → View profile → Copy ID
    "@marcus": "U05E6F7G8H9"

platform-reliability:
  owner: "@marcus"
  leads:
    - "@jordan"
  contributors:
    - "@alex"
    - "@devon"
  notify:
    - "@marcus"
  slack_ids:
    "@marcus": "U05E6F7G8H9"
    "@jordan": "U06I1J2K3L4"

mobile-launch:
  owner: "@priya"
  leads:
    - "@alex"
  contributors:
    - "@devon"
    - "@casey"
  notify:
    - "@priya"
  slack_ids:
    "@priya": "U07M8N9O0P1"
    "@alex": "U08Q2R3S4T5"
```

---

## B4 — Write your CLAUDE.md

Place at the root of your `serro-diy` repo. This is what every teammate's Claude reads before answering any program question.

```markdown
# Program Memory

This repo is the shared program memory for [Your Org]. Claude reads it before answering
any program question and writes daily digests back to digests/.

## How to answer program questions

1. Read `program_mappings.yaml` to identify the program's owner, charter, and declared sources
2. Read `people_mappings.yaml` to identify contributors and who to attribute signals to
3. Read the most recent file in `digests/` for pre-built context on recent activity
4. If the digest is stale (>24 hours) or the question needs more depth, pull live:
   - GitHub: commits, open PRs, closed PRs in the last 14 days, open issues — declared repos only
   - Slack: messages and threads from the last 14 days — declared channels only
   - Drive: documents in declared folders
5. Synthesize across sources — cite every key claim with a source (PR link, Slack thread, doc name)
6. Flag open blockers and unresolved decisions explicitly

**Never query sources outside the mapping for the active program.**
**Never answer from memory alone — always read the digest first, then pull live if needed.**
**If a source returns nothing, say so — do not silently skip it.**

## Source mapping

All sources are declared in `program_mappings.yaml`. If a channel, repo, or
folder isn't listed there for a program, it is invisible to queries for that program.
If something is missing, open a PR to add it.

## Programs

Program names, owners, and charters are declared in `program_mappings.yaml`.
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

## B6 — How to keep it current

The mapping file and digest are only useful if they stay fresh. There are three ways to do this — pick the one that matches your team's setup. They aren't mutually exclusive; many teams combine the manual owner approach with the automated script.

---

### Option 1 — Dedicated server (automated, lowest ongoing effort)

Run the digest script on a small always-on machine (a $5 VPS, a container on your existing infra, or a GitHub Actions scheduled workflow). It pulls from all declared sources nightly and commits the result back to the repo.

**`scripts/update_digest.sh`**:

```bash
#!/usr/bin/env bash
# update_digest.sh — pull daily activity per program and commit to digests/
set -e

DATE=$(date +%Y-%m-%d)
OUTPUT="digests/${DATE}.md"

echo "# Program Digest — ${DATE}" > "$OUTPUT"
echo "" >> "$OUTPUT"

claude --print "
You are updating the program memory for this org. Today is ${DATE}.

Read program_mappings.yaml (which includes owner, charter, and sources for each program).

For each program in the mapping:
1. Search GitHub for commits, merged PRs, and opened issues in the last 24 hours
   in the repos declared for that program only
2. Search Slack for messages in the channels declared for that program in the last 24 hours
3. Search Google Drive for meeting notes or docs updated today in the declared folders

Write the digest in this format:

## [program-name]
**GitHub**: [what shipped or changed — PR titles, numbers, authors]
**Slack**: [key decisions, blockers, or threads worth noting]
**Meetings**: [any meetings today and outcomes]
**Status**: [one sentence on where this program stands right now]
**Watch**: [open blockers or unresolved decisions]

Write only what actually happened. If there was no activity on a program today, write 'No activity.'
" >> "$OUTPUT"

git add "$OUTPUT"
git commit -m "digest: ${DATE}"
git push
```

Schedule with cron on your server:

```bash
# 6am daily
0 6 * * * cd /path/to/serro-diy && ./scripts/update_digest.sh >> logs/digest.log 2>&1
```

Or as a GitHub Actions workflow (no server needed):

```yaml
# .github/workflows/daily-digest.yml
name: Daily Program Digest
on:
  schedule:
    - cron: '0 6 * * *'
  workflow_dispatch:

jobs:
  digest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code
      - name: Install MCP servers
        run: |
          npm install -g @modelcontextprotocol/server-github
          npm install -g @modelcontextprotocol/server-slack
          npm install -g @modelcontextprotocol/server-gdrive
          claude mcp add github -- npx -y @modelcontextprotocol/server-github
          claude mcp add slack -- npx -y @modelcontextprotocol/server-slack
          claude mcp add gdrive -- npx -y @modelcontextprotocol/server-gdrive
        env:
          GITHUB_PERSONAL_ACCESS_TOKEN: ${{ secrets.GITHUB_PERSONAL_ACCESS_TOKEN }}
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      - name: Run digest update
        run: ./scripts/update_digest.sh
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_PERSONAL_ACCESS_TOKEN: ${{ secrets.GITHUB_PERSONAL_ACCESS_TOKEN }}
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      - name: Commit digest
        run: |
          git config user.name "serro-diy-bot"
          git config user.email "bot@your-org.com"
          git add digests/
          git commit -m "digest: $(date +%Y-%m-%d)" || echo "No changes to commit"
          git push
```

> **When Anthropic ships Workflows**: replace the cron entirely — run the digest as a cloud workflow every few hours instead of once a day. Same script, event-driven trigger.

---

### Option 2 — Run it locally (low setup cost, slow)

If you don't want a server, run the script from your own machine. It works — but expect it to take several minutes per run. The script has to authenticate, fire MCP calls across all programs sequentially, wait for Claude to synthesize, and commit back. On a 5-program org with 4–5 sources per program, that's easily 10–15 minutes.

```bash
cd ~/serro-diy
./scripts/update_digest.sh
```

Fine for occasional manual refreshes or testing before setting up the server. Not reliable as your daily update mechanism if anyone on your team forgets to run it.

---

### Option 3 — Manual updates by a TPM or program owner (no script, no server)

If your team has dedicated TPMs, or if you're a manager directly responsible for a program, you don't need the script at all. You own the mapping — keep it current the same way you'd maintain a wiki.

**What this looks like in practice:**

When you spin up a new program, or when a channel gets renamed, or when a new repo is created — you open a PR to `program_mappings.yaml` directly in GitHub. The PR is the audit trail. Reviews are cheap — it's a few lines of yaml. Merge it, and Claude's next query picks up the new source automatically.

```yaml
# program_mappings.yaml
# Owner: @sarah (TPM) — open a PR to add or change sources

mobile-launch:
  github:
    repos:
      - org/ios
      - org/android
      - org/mobile-api    # added when mobile API repo was created
  slack:
    channels:
      - mobile
      - mobile-launch
      - design
      - mobile-qa         # added when QA spun up their own channel
```

This is the right choice when:
- You have a TPM who already does this kind of operational housekeeping
- Your programs are stable and sources don't change often
- You want explicit control over what's in the mapping (no surprise auto-discovery)

The tradeoff: if nobody opens the PR, the mapping silently drifts. Combine with the health check (below) to catch stale entries.

---

## B7 — Set up the mapping health check

Family B fails silently when the mapping is stale. A health check catches the most common failure mode: a channel gets renamed, a repo gets archived, or a folder gets moved.

**Option A — Shell script + cron** ⚙️

```bash
#!/bin/bash
# check_mapping_health.sh
# Reads program_mappings.yaml and verifies each declared source exists
# Requires: yq (yaml parser), GitHub CLI (gh), SLACK_BOT_TOKEN env var

MAPPING="program_mappings.yaml"
ERRORS=0

# Check GitHub repos
for repo in $(yq '.*.github.repos[]' $MAPPING 2>/dev/null | tr -d '"'); do
  if ! gh repo view "$repo" &>/dev/null; then
    echo "ERROR: GitHub repo not found or inaccessible: $repo"
    ERRORS=$((ERRORS + 1))
  fi
done

# Check Slack channels
if [ -n "$SLACK_BOT_TOKEN" ]; then
  # Fetch all channel names from workspace once (avoids per-channel API calls)
  all_channels=$(curl -sf "https://slack.com/api/conversations.list?limit=1000&exclude_archived=true" \
    -H "Authorization: Bearer $SLACK_BOT_TOKEN" | jq -r '.channels[].name' 2>/dev/null)
  for channel in $(yq '.*.slack.channels[]' $MAPPING 2>/dev/null | tr -d '"'); do
    if ! echo "$all_channels" | grep -qx "$channel"; then
      echo "ERROR: Slack channel not found or archived: #$channel"
      ERRORS=$((ERRORS + 1))
    fi
  done
else
  echo "SKIP: SLACK_BOT_TOKEN not set — skipping Slack channel checks"
fi

if [ $ERRORS -gt 0 ]; then
  echo "$ERRORS source(s) missing. Update program_mappings.yaml."
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
Read program_mappings.yaml. For each declared GitHub repo, Slack channel,
and Drive folder, verify it still exists and is accessible. Report any that are missing
or inaccessible. Post the result to #program-ops.
```

Schedule this as a `/schedule` command or cron-triggered Claude invocation.

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
