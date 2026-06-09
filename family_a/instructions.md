# Family A: Full Context Pull — Instructions

> For orgs with ≤5 engineers, 1 repo, 1–2 Slack channels, and 1–2 active programs. No config, no mapping, no ingestion. Claude pulls everything at query time.

---

## Steps

- [A1 — Connect your tools](#a1--connect-your-tools)
- [A2 — Name your programs](#a2--name-your-programs)
- [A3 — Write your CLAUDE.md](#a3--write-your-claudemd)
- [A4 — Run your first query](#a4--run-your-first-query)
- [A5 — Set a growth trigger](#a5--set-a-growth-trigger)

---

## A1 — Connect your tools

Install MCP servers for every source your programs touch. For most micro-orgs this means GitHub, Slack, and Google Drive. Add transcripts if you use Google Meet or Notion.

**GitHub MCP**

```bash
claude mcp add github --transport http https://api.githubcopilot.com/mcp/
```

Requires a GitHub personal access token with `repo` and `read:org` scopes. Set it as `GITHUB_PERSONAL_ACCESS_TOKEN` in your environment.

**Slack MCP**

```bash
claude mcp add slack --transport http https://mcp.slack.com/
```

Requires a Slack app with `channels:history`, `channels:read`, `users:read` OAuth scopes installed to your workspace.

**Google Drive MCP**

```bash
claude mcp add gdrive --transport http https://mcp.gdrive.com/
```

Requires OAuth credentials with `drive.readonly` scope.

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

## A2 — Name your programs

Write a short file at the root of your repo (or anywhere Claude can find it) that names your active programs. No yaml, no schema — just names and one-line descriptions.

**Example: `programs.md`**

```markdown
# Active Programs

## Auth Modernization
Replace legacy session tokens with JWT. Owner: @sarah. Q3 target.

## Platform Reliability
Reduce p99 latency below 200ms across all API endpoints. Owner: @marcus.
```

This file is what Claude will reference when you ask program-level questions. Keep it short and current.

---

## A3 — Write your CLAUDE.md

`CLAUDE.md` tells Claude what sources belong to this org and how to approach queries. For Family A, it should instruct Claude to pull all sources at query time — no filtering.

**Place at repo root: `CLAUDE.md`**

```markdown
# Program Memory — Claude Instructions

## Sources

Pull all of these when answering program-level questions:

**GitHub**
- Repos: [list your repos, e.g. org/backend, org/frontend]
- Branches: main (recent commits and PRs)

**Slack**
- Channels: [list your channels, e.g. #eng, #platform, #auth-migration]
- History: last 90 days unless the question specifies otherwise

**Google Drive**
- Folders: [list your shared folders, e.g. "Engineering / RFCs", "Program Docs"]

**Programs**
See `programs.md` for the list of active programs and owners.

## How to answer program questions

When asked about a program:
1. Read `programs.md` to identify the program scope
2. Search GitHub for recent PRs, commits, and issues mentioning that program's keywords
3. Search Slack for recent threads in relevant channels
4. Search Drive for relevant docs
5. Synthesize across all sources — note the source of each key claim

Do not answer from memory alone. Always pull fresh context.
```

---

## A4 — Run your first query

Test a real cross-source question. This is your baseline before anything grows.

**Suggested test queries:**

```
What's the current status of [program name]?
What decisions were made about [program name] in the last 2 weeks?
Who has been most active on [program name] recently?
Are there any open blockers on [program name]?
```

**What to check:**
- Does Claude pull from all three sources (GitHub, Slack, Drive)?
- Are the answers accurate against what you know to be true?
- How long does it take? (note the latency)
- Does the response cite its sources?

If the answer looks right and sources are cited — Family A is working. If Claude is missing obvious context, check that the MCP servers are connected and that your `CLAUDE.md` source list is complete.

---

## A5 — Set a growth trigger

Family A breaks as your org grows. Define now what signal tells you to upgrade to Family B, so you don't wait until queries are noticeably degraded.

**Upgrade to Family B when any of these are true:**

| Signal | Threshold |
|---|---|
| Source count | More than 20 sources total (repos + channels + folders) |
| Context window | Claude returns truncated results or says it can't fit all sources |
| Latency | Cross-source queries take more than 30 seconds |
| Teams | More than one team — sources need to be scoped per program |
| Programs | More than 3 active programs — full-pull becomes noisy |

Write your specific thresholds into `programs.md` so whoever is watching for them knows what to look for.

**When you hit the trigger:** see [`family_b/instructions.md`](../family_b/instructions.md).
