# throughline

> Keep the thread across Claude Code sessions.

Two complementary Claude Code skills:

- **`diary`** — saves a past-tense session log to your Obsidian vault as either a project note or a dated session note.
- **`recall`** — freezes full session context (decisions, open questions, files touched, resume instructions) so you can pause work in one session and resume it cold in another.

`diary` is your project history. `recall` is your handoff brief. Together they keep the throughline of your work intact across every new context window.

---

## Prerequisites

| Skill | Required | Notes |
|---|---|---|
| **diary** | [Obsidian](https://obsidian.md) vault on disk | The skill writes Markdown files into your vault. It will auto-discover your vault on first use. |
| **recall** | Claude Code | Obsidian is **optional** — recall stores its own snapshots in `~/.claude/recall/`. If you have an Obsidian vault, recall will also link to related session notes. |

Both skills require [Claude Code](https://claude.ai/code).

**No manual path configuration needed.** Both skills auto-discover your Obsidian vault at first use by scanning for `.obsidian` directories. Discovered paths are cached in `~/.claude/throughline/config.md`.

---

## Install

```bash
# 1. Clone this repo
git clone https://github.com/chewkaah/throughline.git /tmp/throughline

# 2. Copy the skill folders into your global Claude skills directory
mkdir -p ~/.claude/skills
cp -R /tmp/throughline/diary  ~/.claude/skills/
cp -R /tmp/throughline/recall ~/.claude/skills/

# 3. Restart Claude Code (or start a new session). Verify with:
#      "list available skills"
#    You should see `diary` and `recall` in the list.
```

That's it. No path editing required — the skills will ask about your vault on first use if they can't find it automatically.

---

## Install (for an LLM agent)

If you are a coding agent (Claude Code, etc.) installing this for a user, run these steps in order.

### Step 1 — Verify prerequisites

1. Confirm Claude Code is running and `~/.claude/skills/` is the active global skill directory. If it does not exist, create it: `mkdir -p ~/.claude/skills`.
2. Ask the user: **"Do you use Obsidian?"**
   - If **no** → only install `recall`. The `diary` skill requires a vault.
   - If **yes** → proceed with both. The skills will discover the vault path automatically on first use.

### Step 2 — Fetch the skills

```bash
git clone https://github.com/chewkaah/throughline.git /tmp/throughline
```

### Step 3 — Copy into the skills directory

```bash
cp -R /tmp/throughline/recall ~/.claude/skills/

# diary only if the user uses Obsidian
cp -R /tmp/throughline/diary ~/.claude/skills/
```

### Step 4 — Verify install

Tell the user to run `/help` or ask Claude Code "list available skills" in a new session. Both skills should appear with their descriptions:

- `diary — Save or append a session log to the Obsidian daily session note...`
- `recall — Capture or restore a full session-context snapshot...`

If they don't appear, check that the `name:` field in each `SKILL.md` frontmatter matches the folder name (`diary` and `recall`) and that Claude Code was restarted.

### Step 5 — Smoke test

Have the user invoke each skill once:

- **diary**: say `save to diary — testing install`. On first run, the skill will locate (or ask for) the vault, then write a session note and report the path.
- **recall**: say `recall save smoke-test`. The skill should write `~/.claude/recall/latest.md` and `~/.claude/recall/smoke-test.md`. Then say `recall smoke-test` in a new session — the prior context should reload.

If the smoke test fails, the most common issues are:
1. Vault not found — the skill will ask the user for the path; provide it and the config is cached for future runs.
2. Claude Code needs a restart to pick up new skills.

---

## How they work

### `diary` — usage

Trigger phrases: `save to diary`, `add to diary`, `save session`, `log session`.

On first use, the skill discovers your Obsidian vault (via `.obsidian` directory scan) and detects your sessions and projects folders. The result is cached at `~/.claude/throughline/config.md` — delete this file to re-run discovery.

The skill routes between two modes:

- **Project route** — appends a dated `### YYYY-MM-DD — <Title>` section under `## Progress Log` in the project's doc. Creates the doc if it doesn't exist yet.
- **Session route** — writes (or appends to) a dated session file for one-off / cross-project / infra work.

### `recall` — usage

Two modes, picked from the trigger phrase:

| You say | Mode | Effect |
|---|---|---|
| `recall save` | save | writes `~/.claude/recall/latest.md` |
| `recall save <slug>` | save | writes both `latest.md` and `<slug>.md` |
| `freeze context [<slug>]` | save | same as above |
| `recall` | load | reads `latest.md` and adopts it as current context |
| `recall <slug>` | load | reads `<slug>.md` and adopts it as current context |
| `resume from recall [<slug>]` | load | same as load |

A **save** distills the current session into TL;DR, where-we-are, locked decisions, open questions, files touched, and resume instructions. A **load** reads that snapshot back in a new session and presents it inline so the next Claude can pick up cold without asking "what were we doing."

If you also use Obsidian, recall will link related session notes it finds in your vault.

### How they fit together

| When you... | Use |
|---|---|
| Finish a chunk of project work and want a permanent log | `save to diary` |
| Need to pause work mid-session and resume later | `recall save <slug>` |
| Want to resume a paused session | `recall <slug>` |
| Both — wrap up cleanly AND leave a handoff brief | `save to diary` then `recall save <slug>` |

---

## Repo layout

```
.
├── README.md
├── LICENSE
├── diary/
│   └── SKILL.md
└── recall/
    └── SKILL.md
```

Each skill is a single `SKILL.md` file — no scripts, no dependencies beyond what's described above.

---

## License

MIT. See `LICENSE`.
