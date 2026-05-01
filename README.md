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
| **diary** | [Obsidian](https://obsidian.md) vault on disk | The skill writes Markdown files into your vault folders. Without a vault, the skill has nothing to write to. |
| **recall** | Claude Code | Obsidian is **optional** — recall stores its own snapshots in `~/.claude/recall/`. If you have an Obsidian vault, recall will also link to related session notes. |

Both skills assume you are running [Claude Code](https://claude.ai/code) and that `~/.claude/skills/` is the global skill directory.

### Vault folder layout (diary only)

The diary skill expects two folders inside your Obsidian vault:

```
<VAULT_PATH>/
├── 08-Projects/      # one folder per project, each with a project doc
└── 09-Sessions/      # dated session notes (YYYY-MM-DD-<slug>.md)
```

If you use a different folder layout, edit the paths inside `diary/SKILL.md` to match.

---

## Install (for a human)

```bash
# 1. Clone this repo somewhere temporary
git clone https://github.com/chewkaah/throughline.git /tmp/throughline

# 2. Copy the two skill folders into your global Claude skills directory
mkdir -p ~/.claude/skills
cp -R /tmp/throughline/diary  ~/.claude/skills/
cp -R /tmp/throughline/recall ~/.claude/skills/

# 3. Edit ~/.claude/skills/diary/SKILL.md and replace every <VAULT_PATH>
#    with the absolute path to your Obsidian vault root, e.g.
#      /Users/yourname/Documents/Obsidian Vault
#    or an iCloud Drive path like
#      /Users/yourname/Library/Mobile Documents/com~apple~CloudDocs/Documents/Obsidian Vault
#
#    Do the same in ~/.claude/skills/recall/SKILL.md if you want recall
#    to find related Obsidian session notes. Skip this if you don't use Obsidian.

# 4. Restart Claude Code (or start a new session). Verify with:
#      "list available skills"
#    You should see `diary` and `recall` in the list.
```

---

## Install (for an LLM agent)

If you are a coding agent (Claude Code, etc.) installing this for a user, run these steps in order. Stop and ask the user only at the marked checkpoints.

### Step 1 — Verify prerequisites

1. Confirm Claude Code is running and `~/.claude/skills/` is the active global skill directory. If `~/.claude/skills/` does not exist, create it: `mkdir -p ~/.claude/skills`.
2. Ask the user: **"Do you use Obsidian?"**
   - If **no** → only install `recall`. The `diary` skill is non-functional without a vault. Skip to Step 4 with `INSTALL_DIARY=false`.
   - If **yes** → ask: **"What is the absolute path to your Obsidian vault root?"** Store the answer as `VAULT_PATH`. Verify the path exists with `test -d "$VAULT_PATH" && echo OK`.
3. If `INSTALL_DIARY=true`, verify the expected vault subfolders exist:
   ```bash
   test -d "$VAULT_PATH/08-Projects" && echo "projects-ok"
   test -d "$VAULT_PATH/09-Sessions" && echo "sessions-ok"
   ```
   If either is missing, ask the user whether to create them (`mkdir -p "$VAULT_PATH/08-Projects" "$VAULT_PATH/09-Sessions"`) or whether they use a different layout (in which case you will need to edit the path references inside `diary/SKILL.md` to match).

### Step 2 — Fetch the skills

```bash
git clone https://github.com/chewkaah/throughline.git /tmp/throughline
```

### Step 3 — Copy into the skills directory

```bash
# recall is always installed
cp -R /tmp/throughline/recall ~/.claude/skills/

# diary only if INSTALL_DIARY=true
cp -R /tmp/throughline/diary ~/.claude/skills/
```

### Step 4 — Substitute `<VAULT_PATH>` placeholders

The skills ship with the literal string `<VAULT_PATH>` everywhere a vault path appears. Replace it with the user's actual path. macOS / Linux:

```bash
# diary (only if installed)
sed -i.bak "s|<VAULT_PATH>|$VAULT_PATH|g" ~/.claude/skills/diary/SKILL.md && rm ~/.claude/skills/diary/SKILL.md.bak

# recall (only if the user uses Obsidian; otherwise leave placeholders — the skill works without)
sed -i.bak "s|<VAULT_PATH>|$VAULT_PATH|g" ~/.claude/skills/recall/SKILL.md && rm ~/.claude/skills/recall/SKILL.md.bak
```

Verify no `<VAULT_PATH>` placeholders remain in installed skills the user wanted vault-wired:

```bash
grep -n "<VAULT_PATH>" ~/.claude/skills/diary/SKILL.md  ~/.claude/skills/recall/SKILL.md 2>/dev/null
```

(Empty output = success for installed skills.)

### Step 5 — Verify install

Tell the user to run `/help` or ask Claude Code "list available skills" in a new session. Both skills should appear with their descriptions:

- `diary — Save or append a session log to the Obsidian daily session note...`
- `recall — Capture or restore a full session-context snapshot...`

If they don't appear, check that the `name:` field in each `SKILL.md` frontmatter matches the folder name (`diary` and `recall`) and that Claude Code was restarted.

### Step 6 — Smoke test

Have the user invoke each skill once:

- **diary**: open a chat and say `save to diary — testing install`. The skill should write to `<VAULT_PATH>/09-Sessions/YYYY-MM-DD-<slug>.md` and report the path back.
- **recall**: in any chat say `recall save smoke-test`. The skill should write `~/.claude/recall/latest.md` and `~/.claude/recall/smoke-test.md`. Then say `recall smoke-test` in a new session — the prior context should be reloaded.

If the smoke test fails, the most common issues are:
1. `<VAULT_PATH>` placeholder not substituted (re-run Step 4).
2. Vault folders `08-Projects/` or `09-Sessions/` don't exist (create them).
3. Claude Code needs a restart to pick up new skills.

---

## How they work

### `diary` — usage

Trigger phrases: `save to diary`, `add to diary`, `save session`, `log session`.

The skill decides between two routes:

- **Project route** — appends a dated `### YYYY-MM-DD — <Title>` section under `## Progress Log` in the project's doc inside `08-Projects/`. Creates the doc if it doesn't exist yet.
- **Session route** — writes (or appends to) `09-Sessions/YYYY-MM-DD-<slug>.md` for one-off / cross-project / infra work that doesn't belong to a single named project.

It picks based on whether a project doc already exists and whether the work is feature-level or infra-level. Read `diary/SKILL.md` for the full routing rules.

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

A **save** distills the current session into TL;DR, where-we-are, locked decisions, open questions, files touched, and resume instructions. A **load** reads that snapshot back in a new session and presents it inline as the working context, so the next Claude can pick up cold without asking "what were we doing".

If you also use Obsidian, recall will link related session notes from `<VAULT_PATH>/09-Sessions/` so the load-mode reader can pull them in too.

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
