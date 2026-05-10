---
name: recall
description: Capture or restore a full session-context snapshot so work can pause in one Claude Code session and resume cold in another. Use when the user says "recall save", "save recall", "freeze context" (save mode — captures TL;DR, where-we-are, locked decisions, open questions, files touched, external refs, related Obsidian session notes, and resume instructions to ~/.claude/recall/), or "recall", "recall <slug>", "resume from recall", "load recall" (load mode — reads the snapshot, finds the related Obsidian session note, and presents both inline as the new session's working context). Different from claude-mem (auto observations) and the diary skill (past-tense session log) — recall is heavyweight, intentional, complete, designed for cold-start resumption.
license: MIT
metadata:
  version: 1.1.0
---

# Recall

You are the recall skill. You have two modes — **save** and **load** — determined by what the user said when invoking you.

## Obsidian Integration (Optional)

Recall can optionally cross-reference Obsidian session notes. To use this, the skill will auto-discover your vault at runtime. If you don't use Obsidian, the skill works without it — all Obsidian steps are skipped gracefully.

### Vault Discovery

Do NOT assume or hardcode any vault path. If Obsidian integration is needed (finding related notes), resolve the vault path at runtime:

**Check for cached config first:**

```bash
cat ~/.claude/throughline/config.md 2>/dev/null
```

If a config exists with a `vault` key, use it. If not, discover:

```bash
find ~ -maxdepth 6 -name ".obsidian" -type d 2>/dev/null
```

- **Zero results**: Obsidian not found — skip all Obsidian steps silently, leave `related-obsidian-notes` empty.
- **One result**: Use its parent as `vault`. No confirmation needed for a save/load operation.
- **Multiple results**: Use the first one OR check if the user has a config at `~/.claude/throughline/config.md`. If still ambiguous, skip Obsidian steps and note it in output: "Multiple vaults found — Obsidian notes skipped. Run `diary` to configure a default vault."

When a vault is resolved, also read the `sessions_dir` and `projects_dir` from the config (defaults: look for a folder named like `*Sessions*` or `*Journal*`).

---

## Mode detection

Look at the user's invocation message to determine mode:

| User said | Mode | Optional slug |
|---|---|---|
| `recall save` | save | none → writes to `latest.md` only |
| `recall save <slug>` | save | `<slug>` → writes to both `latest.md` and `<slug>.md` |
| `save recall` / `save recall <slug>` | save | same as above |
| `freeze context` / `freeze context <slug>` | save | same as above |
| `recall` | load | none → reads `latest.md` |
| `recall <slug>` | load | `<slug>` → reads `<slug>.md` |
| `resume from recall` / `resume from recall <slug>` | load | same as above |
| `load recall` / `load recall <slug>` | load | same as above |

If ambiguous, default to **load** mode with no slug (read `latest.md`).

---

## Storage layout

```
~/.claude/recall/
├── latest.md              # Always the most recent unnamed save (overwritten each save)
└── <slug>.md              # Optional named saves, persistent
```

Slugs are kebab-case (e.g. `feature-wiring`, `launch-funnel`).

---

## SAVE MODE workflow

When invoked in save mode:

### 1. Determine slug

If the user provided a slug, use it. If not, no slug (just write `latest.md`).

### 2. Synthesize the recall content

Build a markdown doc from the current session's working state. Don't dump the transcript — distill it into the schema below. Use the user's actual decisions, files touched, refs, etc.

```markdown
---
name: <slug or "latest">
date: <ISO timestamp>
session-topic: <one-line description>
project: <inferred from context>
tags:
  - recall
  - <topic-relevant tags>
related-obsidian-notes:
  - <path/to/related-note.md>
  - ...
related-tickets:
  - <Linear IDs, GH issues, etc.>
---

# Recall: <session topic>

## TL;DR

<1-3 sentences capturing the essence of the work>

## Where we are

<Current state — what's been done, what's in flight, what's blocked, what's the next concrete action>

## Decisions made (locked)

<Bullet list of decisions that should NOT be re-litigated. Each with one-sentence rationale if non-obvious.>

- ...

## Open questions

<Bullet list of unresolved questions. Each tagged with who's blocking on what.>

- ...

## Files modified this session

| Path | Change |
|---|---|
| ... | ... |

## Files created this session

| Path | Purpose |
|---|---|
| ... | ... |

## External refs

- Linear tickets: <list>
- URLs: <list>
- Other tools: <list>

## Conversation arc (key turns)

<Compact narrative — 5-10 bullet points — of how we got here. Not a transcript. Just the load-bearing turns.>

## Resume instructions for next session

<The most important field. What should the next Claude do FIRST when this is loaded? Be specific:
- "Read X, Y, Z files first"
- "The user's next ask will likely be ..."
- "Don't re-do A, B, C — those are locked"
- "Open question is W — ask the user about it before anything else"
>
```

### 3. Find related Obsidian session notes (optional)

Run vault discovery (see above). If a vault is found, search for session notes that relate to the current work. Look in `<vault>/<sessions_dir>/` and `<vault>/<projects_dir>/<project>/`. Match by:
- Project name in path
- Recent dates (last 7 days)
- Tags / topic overlap

Include matching paths in the `related-obsidian-notes` frontmatter. Don't inline file contents — just paths. Load mode handles re-reading.

If no vault is found or Obsidian isn't configured, leave `related-obsidian-notes` empty and continue.

### 4. Write the file(s)

- Always write `~/.claude/recall/latest.md` (overwriting any existing version)
- If slug was provided, ALSO write `~/.claude/recall/<slug>.md`

### 5. Confirm to the user

Report:
- Path(s) written
- Whether a slug was used
- Number of related Obsidian notes found (or "Obsidian not configured" if skipped)
- Brief summary of what was captured (use the TL;DR field)

---

## LOAD MODE workflow

When invoked in load mode:

### 1. Determine which file to load

If the user provided a slug, load `~/.claude/recall/<slug>.md`. If not, load `~/.claude/recall/latest.md`.

If the file doesn't exist, tell the user clearly and list what IS available in `~/.claude/recall/` so they can pick a different slug.

### 2. Read the recall file

Parse the frontmatter and body.

### 3. Find and read the most recent related Obsidian session note (optional)

Run vault discovery (see above). If a vault is found:

Look at the `related-obsidian-notes` field in the frontmatter. Find the most recent one (by date in filename or by file mtime). Read its contents.

If no `related-obsidian-notes` field, search for a recent session note matching the project tag from frontmatter:
- Look in `<vault>/<sessions_dir>/` for files matching `YYYY-MM-DD-*<project-slug>*.md` from the last 7 days
- Look in `<vault>/<projects_dir>/<project>/` for `Overview.md` or similar

Read the most recent matching file.

If no vault is found, skip this step silently.

### 4. Present the loaded context inline

In your response to the user, present:

```
## Recall loaded: <session-topic>

**From:** ~/.claude/recall/<file>
**Captured:** <date>
**Project:** <project>

### TL;DR
<from recall file>

### Where we left off
<from recall file>

### Locked decisions
<from recall file>

### Open questions
<from recall file>

### Resume instructions
<from recall file>

### Related Obsidian session note: <path>
<inline excerpt or summary of the relevant session note>

### Files to be aware of
<list from recall file with brief context per file>
```

If no Obsidian note was found or Obsidian isn't configured, omit the "Related Obsidian session note" section entirely.

### 5. Adopt the context

After presenting, treat the loaded context as the working state for the rest of the session. The user's next message picks up from where the prior session left off.

End your response with: "**Loaded. Ready to continue from where we left off.** What's the next move?"

---

## Reference handling rules (applies to both modes)

- **Don't inline file contents** in the recall doc body — use paths + brief context. The new session can re-read the actual files when needed.
- **DO inline short critical excerpts** (locked specs, single-line decisions, exact credentials, etc.) directly into the recall doc — these need to survive even if the referenced file changes.
- **Keep the recall doc opinionated** — it's a handoff brief, not a transcript dump. Distill, don't transcribe.

---

## Working with claude-mem and the diary skill

Recall is intentionally different from these:

- **claude-mem** captures fuzzy auto-observations cross-session — useful for "did we ever solve X." Don't duplicate that work; recall is for full intentional state snapshots.
- **diary** writes past-tense session logs to Obsidian — useful for project history. Recall doesn't replace it; the two are complementary. A normal pause might use both: `recall save` + `save to diary`.

When in save mode, if a session note exists for today (in the discovered vault), list it under `related-obsidian-notes` so the load-mode reader picks it up automatically.

---

## Edge cases

- **No prior recall to load** — tell the user clearly: "No recall found at <path>. Available recalls: <list>." Don't make up content.
- **Multiple slugs match a partial input** — list matches and ask the user to pick.
- **Stale recall (older than 30 days)** — load it but flag the staleness at the top: "⚠️ This recall is N days old. State may have drifted."
- **Recall references files that no longer exist** — load anyway, but note which references are dead at the bottom of the load output so the user knows.
- **Obsidian not found** — continue without it; never error or block on missing Obsidian integration.

---

## Quality bar

A good recall save:
- The next Claude can pick up the work without asking obvious "what were we doing" questions
- Locked decisions are captured so they don't get re-litigated
- Open questions are explicit so the new session knows what's unresolved
- Resume instructions are specific enough to act on immediately

A bad recall save:
- Verbose transcript dumps instead of distilled state
- Vague "we worked on the project" without specifics
- Missing the "next concrete action" — leaves the new session adrift
- Inlines giant file contents that bloat context
