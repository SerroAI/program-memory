# Family A: Full Context Pull — Instructions

> For orgs with ≤5 engineers, 1 repo, 1–2 Slack channels, and 1–2 active programs. No config, no mapping, no ingestion. Claude pulls everything at query time.

---

## Steps

- [A1 — Connect your tools](#a1--connect-your-tools)
- [A2 — Create a shared serro-diy repo](#a2--create-a-shared-serro-diy-repo)
- [A3 — Write program_mappings.yaml, people_mappings.yaml, and CLAUDE.md](#a3--write-program_mappingsyaml-people_mappingsyaml-and-claudemd)
- [A4 — Add the daily digest script](#a4--add-the-daily-digest-script)
- [A5 — Run your first query](#a5--run-your-first-query)

---

## A1 — Connect your tools

Install MCP servers for every source your programs touch. For most micro-orgs this means GitHub, Slack, and Google Drive. Add transcripts if you use Google Meet or Notion.

**GitHub MCP**

```bash
claude mcp add github -- npx -y @modelcontextprotocol/server-github
```

Set `GITHUB_PERSONAL_ACCESS_TOKEN` in your environment. Token needs `repo` and `read:org` scopes.

**Slack MCP**

```bash
claude mcp add slack -- npx -y @modelcontextprotocol/server-slack
```

Set `SLACK_BOT_TOKEN` in your environment. Slack app needs `channels:history`, `channels:read`, `users:read` OAuth scopes installed to your workspace.

**Google Drive MCP**

```bash
claude mcp add gdrive -- npx -y @modelcontextprotocol/server-gdrive
```

Runs a local OAuth flow on first use — follow the prompts in your terminal.

> MCP server packages are maintained at [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers). If a command fails, check there for the current package name or installation method.

**Transcripts** ⚙️ — pick one that fits your setup:
- Google Meet: use the Drive MCP to access auto-generated transcripts stored in Drive
- Notion: Notion MCP if your meeting notes live there
- Plain markdown files in a repo: GitHub MCP already covers this

**Verify all connections are working:**

```
/mcp
```

All servers should show as connected before proceeding.

---

## A2 — Create a shared serro-diy repo

Before writing any files, create a dedicated repo for your program memory. This repo is the single place Claude reads from — and eventually writes back to. Every teammate points their Claude Code at it.

```bash
# On GitHub: create a new repo named serro-diy (or org-memory, team-brain, etc.)
# Then clone it locally
git clone https://github.com/your-org/serro-diy
cd serro-diy
```

Why a separate repo and not a folder in your main codebase:
- All teammates can clone it without access to product code
- Claude can commit daily digest updates back to it without touching your main repos
- It's the one place everyone knows to look

Create the initial structure:

```
serro-diy/
  CLAUDE.md               ← instructions for Claude (you'll write this next)
  program_mappings.yaml   ← owner, charter, and sources per program
  people_mappings.yaml    ← contributors, leads, and Slack IDs per program
  digests/                ← daily digest files written by the update script
```

### Connecting teammates to this repo

`CLAUDE.md` is only active when Claude is reading it. Without pointing Claude at this repo, the memory is just a file sitting on disk — Claude won't know it exists and won't check it.

Every teammate connects once. Two ways to do it:

**Add it as a project in Claude Code**

From inside any repo (your product code, wherever you normally work):

```
/project:add ~/path/to/serro-diy
```

Claude loads the `serro-diy` CLAUDE.md alongside your current project. Program questions work from anywhere without switching directories.

For scripting and automation (e.g. the digest script), `cd` into the `serro-diy` repo before invoking `claude` so the right `CLAUDE.md` is loaded.

The `CLAUDE.md` is what makes the mapping and digest active instructions rather than static text. Without it being loaded, Claude answers from its own knowledge — stale and uncited.

---

## A3 — Write program_mappings.yaml, people_mappings.yaml, and CLAUDE.md

**`program_mappings.yaml`** — one entry per active program, with owner, charter, and all sources. This is what Claude reads to scope any question and find the right signals.

```yaml
# program_mappings.yaml

auth-modernization:
  owner: "@sarah"
  charter: "Replace legacy session tokens with JWT across all services. Q3 target."
  github:
    repos:
      - org/auth-service
      - org/backend
  slack:
    channels:
      - auth-migration
  drive:
    folders:
      - "Engineering/Auth RFCs"

platform-reliability:
  owner: "@marcus"
  charter: "Reduce p99 latency below 200ms across all API endpoints. Ongoing."
  github:
    repos:
      - org/api-gateway
      - org/backend
  slack:
    channels:
      - platform
      - incidents
```

**`people_mappings.yaml`** — who has context in each program and who to notify for follow-ups.

```yaml
# people_mappings.yaml

auth-modernization:
  owner: "@sarah"
  contributors:
    - "@alex"
    - "@jordan"
  notify:
    - "@sarah"
  slack_ids:
    "@sarah": "U04A1B2C3D4"

platform-reliability:
  owner: "@marcus"
  contributors:
    - "@devon"
  notify:
    - "@marcus"
  slack_ids:
    "@marcus": "U05E6F7G8H9"
```

**`CLAUDE.md`** — tells Claude how to answer questions and where to look. Place it at the root of the `serro-diy` repo.

```markdown
# Program Memory

This repo is the shared memory for [Your Org]. Claude reads it before answering
any program question and writes daily digests back to digests/.

## How to answer program questions

1. Read `program_mappings.yaml` to identify the program's owner, charter, and sources
2. Read `people_mappings.yaml` to identify contributors and who to attribute signals to
3. Read the most recent file in `digests/` for a pre-built summary of recent activity
4. If the digest is stale (>24 hours) or the question needs more depth, pull fresh from GitHub, Slack, and Drive
5. Synthesize across sources — cite the source of every key claim (PR link, Slack thread, doc title)
6. Flag anything that looks like a blocker or a decision that hasn't been made yet

Never answer from memory alone. Always read `digests/` first, then pull live if needed.

## How to update program memory

The update script (`scripts/update_digest.sh`) runs daily and commits a new file to
`digests/YYYY-MM-DD.md`. If you're asked to update program memory manually, follow
the same format and commit directly to this repo.
```

Once both files are committed and pushed, every teammate with the `serro-diy` repo cloned can point Claude at it:

```
/project:add ~/path/to/serro-diy
```

Or set it as the project directory when starting Claude Code.

---

## A4 — Add the daily digest script

This script runs once a day (or on a cron), pulls yesterday's activity across GitHub, Slack, and meetings, maps it to your programs, and commits a digest file back to the repo. Claude reads this file before answering questions — so queries are fast and don't need to re-pull everything from scratch.

Create `scripts/update_digest.sh` in your `serro-diy` repo:

```bash
#!/usr/bin/env bash
# update_digest.sh — pull daily activity and commit to digests/

set -e

DATE=$(date +%Y-%m-%d)
OUTPUT="digests/${DATE}.md"

echo "# Program Digest — ${DATE}" > "$OUTPUT"
echo "" >> "$OUTPUT"

# Run Claude with all MCPs connected to generate the digest
claude --print "
You are updating the program memory for this org. Today is ${DATE}.

Read program_mappings.yaml to get the list of active programs, their owners, charters, and source mappings.

For each program:
1. Search GitHub for commits, merged PRs, and opened issues in the last 24 hours
   across the repos listed for that program
2. Search Slack for messages in the channels listed for that program from the last 24 hours
3. Search for any meeting transcripts or notes from today in Google Drive

Then write a digest in this format:

## [Program Name]
**GitHub**: [what shipped or changed — PR titles and numbers]
**Slack**: [key decisions, blockers, or threads worth noting]
**Meetings**: [any meetings today and their outcomes]
**Status**: [one sentence on where this program stands right now]
**Watch**: [anything that looks like a blocker or open decision]

Write only what actually happened. If there was no activity on a program today, write 'No activity.'
" >> "$OUTPUT"

git add "$OUTPUT"
git commit -m "digest: ${DATE}"
git push
```

Make it executable:

```bash
chmod +x scripts/update_digest.sh
```

Schedule it with cron (runs at 6am daily):

```bash
crontab -e
# Add:
0 6 * * * cd /path/to/serro-diy && ./scripts/update_digest.sh >> logs/digest.log 2>&1
```

> **When Anthropic ships Workflows**: replace the cron with a cloud workflow that runs every few hours instead of once a day. The script logic stays the same — the trigger becomes event-driven rather than scheduled.

---

## A5 — Run your first query

With the repo cloned, CLAUDE.md in place, and at least one digest committed, test a real cross-source question.

**Suggested test queries:**

```
What's the current status of [program name]?
What decisions were made about [program name] in the last 2 weeks?
Who has been most active on [program name] recently?
Are there any open blockers on [program name]?
```

**What to check:**
- Does Claude read the digest first before going to live sources?
- Are the answers accurate against what you know to be true?
- Does the response cite its sources (PR links, Slack threads, doc names)?
- Is the digest file getting committed daily?

If Claude cites sources and the digest is updating — Family A is working.

**Known limits.** This approach holds until:
- You have more than ~20 sources total and the digest gets too long to be useful
- You need per-program filtering (different teams shouldn't see each other's digests)
- You need code-level understanding (what this PR actually does, not just its title)
- You need temporal reasoning across quarters, not just recent days

When any of those hit, see [Family B](../family_b/instructions.md) or [Family C](../family_c/overview.md).

