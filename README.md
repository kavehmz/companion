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

Each skill is a single self-contained `SKILL.md` file. Load it into your agent however that agent consumes instructions:

- **Claude Code** — drop the file into `.claude/skills/<name>/SKILL.md` (project) or `~/.claude/skills/<name>/SKILL.md` (user-wide).
- **Codex / other agents** — paste the contents into the agent's instruction/rules file (e.g. `AGENTS.md`, system prompt, custom rules).

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
