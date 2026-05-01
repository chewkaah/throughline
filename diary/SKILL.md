---
name: diary
description: Save or append a session log to the Obsidian daily session note. Triggers on "add to diary", "save session", "log session", or "save to diary".
---

# Diary / Save Session Skill

Save a summary of the current session to your Obsidian vault.

## Configuration

Before using, edit the paths below to point at your own Obsidian vault.

```
Sessions:  <VAULT_PATH>/09-Sessions/
Projects:  <VAULT_PATH>/08-Projects/
```

Replace `<VAULT_PATH>` with the absolute path to your Obsidian vault root (e.g. `/Users/yourname/Documents/Obsidian Vault` or an iCloud Drive path).

## Instructions

### Step 1 — Determine Today's Date

Use the current date (from system-reminder context if available, otherwise `date +%Y-%m-%d`).

### Step 2 — Gather Session Content

If the user provided a description or title, use it. Otherwise, synthesise from the conversation:
- What was worked on
- Key decisions made
- Files created or changed
- Problems solved
- Any pending items

### Step 3 — Route: Project Doc or Session Note?

**Ask: is this work primarily on a named project?**

A named project = a client, product, or codebase that has (or should have) a persistent reference document in `08-Projects/` — e.g. a specific client, product, or recurring initiative.

```
Is this work on a named project?
├── YES → go to Step 4A (Project Doc route)
└── NO  → go to Step 4B (Session Note route)
```

**Project signals:**
- Work touches a specific client or product codebase
- Changes are feature/fix-level (not infrastructure or tooling)
- There's already a project folder in `08-Projects/`
- The user says "update the [project] docs/notes"

**Session signals:**
- Infrastructure, agent setup, tooling, dev environment
- Cross-project or meta work
- No specific persistent project to attach it to

---

### Step 4A — Project Doc Route

**Find or create the project document:**

```bash
find "<VAULT_PATH>/08-Projects/" -name "*.md" | xargs grep -l "[project-name]" 2>/dev/null
```

- **If a project doc exists** → append a dated progress section to it (see format below)
- **If no project doc exists** → create one at `08-Projects/[Client-or-Org]/[Project-Name]/[Project-Name].md`

**Appending to an existing project doc — add under a `## Progress Log` heading:**

```markdown
## Progress Log

### YYYY-MM-DD — [Short Title]
- [bullet: what changed]
- [bullet: decisions made]
- [bullet: files changed]
```

If `## Progress Log` doesn't exist yet, add it at the bottom of the file.

**Creating a new project doc** — use this structure:

```markdown
---
tags:
  - project
  - [relevant-tags]
created: 'YYYY-MM-DD'
---
# [Project Name]

## Overview
[1–2 sentences]

## Status
**[Status]** — [brief note]

## Repos & Hosting
| Resource | Details |
|---|---|

## Tech Stack
| Layer | Detail |
|---|---|

## Progress Log

### YYYY-MM-DD — [Short Title]
- [bullet points]
```

---

### Step 4B — Session Note Route

**Check if a session note already exists for today:**

```bash
ls "<VAULT_PATH>/09-Sessions/" | grep "$(date +%Y-%m-%d)"
```

- **If a file exists for today with the same topic slug** → append a new numbered section to it
- **If no file exists for today** → create a new one
- **If files exist for today but cover a different topic** → create a new file with a unique slug

**Slug:** Generate a short kebab-case slug from the main topic (2–5 words). Examples:
- `client-feature-sync`
- `product-stripe-setup`

**New session file format:**

```markdown
---
tags:
  - session
  - [relevant-tags]
created: 'YYYY-MM-DD'
---
# YYYY-MM-DD -- [Session Title]

## Summary

[1–3 sentence overview of the session.]

## What Happened

### 1. [Topic]
- [bullet points]

### 2. [Topic]
- [bullet points]

## Files Changed

| File | Change |
|---|---|
| `path/to/file` | Description |
```

**Appending:** Add a new `### N. [Topic]` section under `## What Happened`, update `## Files Changed`. Never duplicate existing content.

---

### Step 5 — Confirm

After writing, tell the user:
- The file path created or updated
- The section title(s) added
- Whether it was a new file or an append
- Whether it went to a project doc or a session note

## Rules

- Always use the vault paths above — never create notes elsewhere
- Session file naming: `YYYY-MM-DD-[slug].md`
- Keep bullets concise — this is a log, not a report
- Tags should be lowercase, relevant to the work done
- If the user provides a specific title or description, use it verbatim
- Never overwrite existing sections — only append
- When in doubt between routes, prefer the project doc route if a project doc already exists
