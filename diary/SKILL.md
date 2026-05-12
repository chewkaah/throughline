---
name: diary
description: Use when the user asks to save, log, diary, recap, or persist work from the current session into Obsidian
---

# Diary / Save Session Skill

Save a concise progress update to the right Obsidian home. Prefer durable canonical, infrastructure, or project docs over throwaway dated session notes.

## Vault Discovery

Do NOT assume or hardcode any vault path. Resolve the vault and folder paths fresh.

### Step D1 - Load or discover vault config

Check for `~/.claude/throughline/config.md`. If it exists, read:

- `vault`: absolute path to vault root
- `sessions_dir`: path to the sessions folder, relative to vault or absolute
- `projects_dir`: path to the projects folder, relative to vault or absolute
- `infrastructure_dir`: optional path to infrastructure/system notes, relative to vault or absolute

If the config file does not exist, discover the vault:

```bash
find ~ -maxdepth 6 -name ".obsidian" -type d 2>/dev/null
```

- **Zero results**: Ask for the vault root path.
- **One result**: Use its parent directory as `vault`; confirm before proceeding.
- **Multiple results**: List them and ask the user to choose.

Once `vault` is confirmed, discover likely folders:

```bash
ls "<vault>"
```

Look for:

- sessions: names containing `session`, `log`, `journal`, or numbered prefixes like `09-`
- projects: names containing `project`, `client`, or numbered prefixes like `08-`
- infrastructure: names containing `infrastructure`, `system`, `ops`, `admin`, or numbered prefixes like `03-`

If a folder is not obvious, ask. Save config:

```markdown
---
vault: /absolute/path/to/vault
sessions_dir: 09-Sessions
projects_dir: 08-Projects
infrastructure_dir: 03-Infrastructure
---
```

If the user has no infrastructure folder, leave `infrastructure_dir` blank and use the projects folder for durable non-session docs.

## Step 1 - Determine Date

Use the current date from context if available. Otherwise run `date +%Y-%m-%d`.

## Step 2 - Synthesize Content

Capture only what helps future work:

- what was worked on
- key decisions
- files created or changed
- problems solved
- pending items and next action

## Step 3 - Route To The Durable Home First

Do not default to the sessions folder. Search existing durable homes first.

```bash
find "<vault>/<infrastructure_dir>" "<vault>/<projects_dir>" -name "*.md" 2>/dev/null | rg -i "topic|project|system|agent|repo|client"
```

Routing priority:

1. **Existing canonical doc**: append/update the relevant section there.
2. **Existing project doc**: append under `## Progress Log`.
3. **Existing infrastructure/system section**: create/update a note there for cross-system automation, agents, skills, env vars, MCPs, permissions, infrastructure, or tooling.
4. **New canonical doc**: create only when the topic is durable and big enough.
5. **Session note**: create only when no durable home exists.

## Canonical Doc Threshold

Create a new canonical doc when at least two are true:

- the topic spans multiple repos, tools, agents, clients, or systems
- future sessions will need a stable entry point
- it defines architecture, operating policy, recurring workflow, or a source of truth
- it has multiple child notes, tickets, or deliverables to index
- the user asks for "canonical", "source of truth", "operating doc", "index", "system", or "automation"

Good filing rules:

- cross-system tooling/agents/automation -> infrastructure/system folder
- product/client/codebase work -> projects folder
- one-off recap/history only -> sessions folder

## Step 4A - Append To Existing Canonical Or Infrastructure Doc

If a relevant doc exists, do not create a new session note. Add a dated section under the most relevant heading.

Preferred headings:

- `## Progress Log`
- `## Audit Log`
- `## Decisions`
- `## Open Items`
- `## Next Build Order`
- `## Filing Notes`

Format:

```markdown
### YYYY-MM-DD - [Short Title]
- [what changed]
- [decision made]
- [next action]
```

If no suitable heading exists, add `## Progress Log` at the bottom.

## Step 4B - Append To Existing Project Doc

Find the project doc under the projects folder. Append under `## Progress Log`:

```markdown
### YYYY-MM-DD - [Short Title]
- [what changed]
- [decision made]
- [files changed]
```

If no `## Progress Log` exists, add it at the bottom.

## Step 4C - Create New Canonical Doc

Use this only when the canonical threshold is met.

```markdown
---
tags:
  - canonical
  - [topic-tags]
created: 'YYYY-MM-DD'
updated: 'YYYY-MM-DD'
status: active
---
# [Topic]

## Overview
[1-3 sentences]

## Current State
- [facts]

## Decisions
- [locked decisions]

## Open Items
- [pending work]

## Related Notes
- [[note]]

## Progress Log

### YYYY-MM-DD - [Short Title]
- [summary]
```

## Step 4D - Create Or Append Session Note

Only use the sessions folder when the work is ephemeral or no durable home exists.

Check for a same-day same-topic note:

```bash
ls "<vault>/<sessions_dir>/" | grep "YYYY-MM-DD"
```

Append if a matching topic note exists. Otherwise create:

```markdown
---
tags:
  - session
  - [topic-tags]
created: 'YYYY-MM-DD'
---
# YYYY-MM-DD -- [Session Title]

## Summary
[1-3 sentences]

## What Happened

### 1. [Topic]
- [bullets]

## Files Changed

| File | Change |
|---|---|
| `path` | Description |
```

## Filing Audit Before Writing

Before writing, state the route in one sentence:

`Filing route: [existing canonical | existing project | infrastructure canonical | new session] because [reason].`

If you create a session note for a topic that has a canonical or project doc, that is a filing bug.

## Confirm

After writing, report:

- file path created or updated
- section added
- whether this was canonical/project/infrastructure/session
- any notes moved or relinked

## Rules

- Always resolve vault paths via Step D1.
- Never hardcode a user-specific vault path.
- Keep bullets concise; this is a log, not a report.
- Tags should be lowercase and relevant.
- If the user provides a specific title or description, use it verbatim.
- Never overwrite existing sections; only append.
