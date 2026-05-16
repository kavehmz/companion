# Cross-AI Companion Skills

Two paired "companion" skills that let one AI coding agent consult another as a peer reviewer / second brain during work on a local project.

- **[`skills/claude-companion/`](./skills/claude-companion/SKILL.md)** — load into **Codex** (or any agent that reads instruction files). Teaches it how to call **Claude** via the `claude` CLI when it needs a second opinion.
- **[`skills/codex-companion/`](./skills/codex-companion/SKILL.md)** — load into **Claude Code** (or any agent that reads instruction files). Teaches it how to call **Codex** via the `codex exec resume` CLI when it needs a second opinion.

The two skills are symmetric. Together they give you a setup where whichever AI you're working with can pull in the other one for a sanity check on hard decisions, stuck debugging, or plan review — without you having to context-switch.

## Why

When an AI agent is helping with non-trivial work, some decisions are subtle, have real tradeoffs, or sit in territory where the agent is uncertain. A second AI as peer reviewer often catches things the first one missed, while keeping the human (you) free for the decisions that genuinely need a human.

These skills codify *when* to call the other agent, *how* to call it (CLI invocation, context handoff), and — critically — *how to report the consultation back to the human* so nothing happens silently.

## Shared design rules

Both skills follow the same principles:

1. **Pass full context every call.** Neither AI sees the other's conversation with the user, so the calling agent must include project state, what it tried, and the exact question on every invocation.
2. **Resume a session for continuity.** Both CLIs support resuming a prior thread in the same project directory, so the consulted AI can build memory across calls within one work-session.
3. **Reflect back to the user.** After every consultation, the calling agent must surface a short summary ("I asked X, the other AI said Y, I'm going to Z") before acting. No silent delegation.
4. **Humans still own the hard calls.** Destructive actions, product/business choices, anything personal or strategic — those go to the **user**, not to the other AI.

## How to install

Each skill is a single self-contained `SKILL.md` file. How you load it depends on the host agent.

### Codex (one-line install from GitHub)

Codex can install a skill directly from this repo's URL. Point it at the skill's directory:

```
https://github.com/kavehmz/companion/tree/main/skills/claude-companion
https://github.com/kavehmz/companion/tree/main/skills/codex-companion
```

The skill lands in `~/.codex/skills/<name>/SKILL.md` and shows up in new Codex CLI sessions automatically. Already-running sessions need a restart to pick up the new skill — the skill list is loaded at session startup.

### Claude Code (manual)

Claude Code reads skills from `~/.claude/skills/<name>/SKILL.md` (user-wide) or `.claude/skills/<name>/SKILL.md` (per-project). The simplest install is clone + symlink:

```bash
git clone https://github.com/kavehmz/companion.git ~/src/companion
ln -s ~/src/companion/skills/codex-companion ~/.claude/skills/codex-companion
# (or claude-companion if you want Claude to consult itself — usually not what you want)
```

Restart any running Claude Code sessions to pick it up.

### Other agents

Paste the contents of the relevant `SKILL.md` into your agent's instruction/rules file (e.g. `AGENTS.md`, system prompt, custom rules).

### Which skill goes where

The skills are cross-tool consultants — install the *other* AI's companion into the agent you're using:

- Using **Codex** day-to-day, want it to consult Claude → install `claude-companion` into Codex.
- Using **Claude Code** day-to-day, want it to consult Codex → install `codex-companion` into Claude Code.

The two skill directories are independent (`~/.codex/skills/` and `~/.claude/skills/`), so installing in one tool does not propagate to the other. If you want a single source of truth, symlink between them.

Both CLIs (`claude` and `codex`) must be installed and reachable for the corresponding skill to work.

## File layout

```
.
├── skills/
│   ├── claude-companion/
│   │   └── SKILL.md      # Instructions for calling Claude
│   └── codex-companion/
│       └── SKILL.md      # Instructions for calling Codex
└── README.md             # This file
```
