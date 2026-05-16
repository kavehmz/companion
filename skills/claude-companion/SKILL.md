---
name: claude-companion
description: Consult Claude (via the `claude` CLI) as a second-brain for complex decisions, brainstorming, peer review of a chosen approach, or when stuck. Maintains a sticky session, always re-provides context, and reflects Claude's reply back to the user.
---

# claude-companion

You (the AI helping the user) have access to **Claude** as a peer reviewer / brainstorming partner via the `claude` CLI. Use it when the work gets hard.

The user has explicitly opted into this: they want you to consult Claude on non-trivial decisions instead of guessing, but they remain the final authority on anything personal, sensitive, or critical.

---

## When to call Claude

Call Claude **before acting** when ANY of these is true:
- You are not confident your planned approach is correct.
- Two or more reasonable approaches exist and the tradeoffs aren't obvious.
- You've tried something twice and it isn't working.
- The next step is hard to reverse (schema change, data migration, infra change, force-push, etc.) and you want a second opinion on the plan.
- A subtle correctness question (concurrency, security, edge cases) that benefits from a second pair of eyes.

**Do NOT call Claude for:**
- Trivial lookups, syntax, well-known APIs — just do it.
- Anything the user themselves must decide: product/business choices, personal preferences, what to name their company, which feature to ship first, etc. → ask the **user**, not Claude.
- Destructive actions. Confirm with the **user**, never with Claude.

---

## How `claude` works (the CLI)

The CLI runs non-interactively in `-p` (print) mode and prints Claude's reply as plain text to stdout. Just show that text to the user.

**Claude has no memory of your conversation with the user** — you must pass context every call.

For continuity across multiple consultations, use `-c` (`--continue`). It picks up the most recent session in the current working directory, and gracefully starts a fresh one if none exists. No session IDs to track, no state files.

> 🔗 **Every call must use `-c`.** `-c` continues *the most recent session* in this CWD. If you slip in a bare `claude -p` mid-thread (e.g. for a tool probe or quick check), that creates a *new* session, and your next `-c` will continue **that** probe — not your original work-session. Treat probes, version checks, and tool discovery as session-affecting: do them once before the first companion call, or fold them into the first call's prompt.

### Every call

```bash
claude -p -c "$(cat <<'EOF'
[CONTEXT]
Project: <one line — what is being built / what repo>
User: <name or role, if relevant>
What the user asked me to do: <the task you're on>
What I've done so far: <concise — bullets are fine>
What I tried and what happened: <only if relevant>
Constraints I know about: <stack, deadlines, user preferences>

[QUESTION FOR YOU]
<the actual thing you want Claude's opinion on — be specific>

[WHAT WOULD BE A GOOD ANSWER]
<e.g. "two options with tradeoffs", "a yes/no with reasoning", "spot the bug">
EOF
)"
```

On follow-ups, the [CONTEXT] block can be a shorter `[UPDATE SINCE LAST CALL]` since Claude remembers the thread — but **always re-state what changed**, because your conversation with the user moved on and Claude didn't see it.

> ⚠️ **CWD matters.** Claude Code scopes sessions by working directory. `-c` only finds sessions started from the same directory. Always invoke `claude` from the project root for the whole work-session. If the user switches projects, the new project gets its own independent thread automatically.

> 🧹 **If you need a clean thread** (user moved to an unrelated topic and you don't want Claude conflating it with the old one), call once *without* `-c` — that starts a new session, which subsequent `-c` calls will then continue.

### Giving Claude tools (only when text-only consultation isn't enough)

By default, `claude -p` runs with **no tools** — Claude can reason, brainstorm, and review based on what you tell it, but cannot run commands, read files, or check live state. That's the right default for consultations: describe the situation in text and ask for advice.

If you genuinely need Claude to inspect or do something (read a specific file, run a test, check live state), pass `--allowedTools` and `--permission-mode auto`:

```bash
claude -p -c --allowedTools 'Read,Bash(npm test)' --permission-mode auto "<prompt>"
claude -p -c --allowedTools 'Bash(date)' --permission-mode auto "<prompt>"
```

**Syntax** (the only thing that's stable across versions):
- A tool name on its own → that tool is allowed: `Read`, `Edit`, `WebFetch`, etc.
- `Bash(<pattern>)` → Bash is allowed only for commands matching the glob: `Bash(git *)`, `Bash(npm test)`, `Bash(date)`.
- Principle of least privilege: narrow first. `Bash(npm test)` is safer than `Bash(npm *)`, which is safer than bare `Bash`. Allow only what the consultation actually needs.

**Discover the current tool names** — don't rely on a hardcoded list, Claude Code's tool set evolves. Run one probe call:

```bash
claude -p 'List the exact names of your built-in tools, one per line, no description, no markdown.'
```

The output is the authoritative list for the installed `claude` version. Cache it for the work-session if you'll need it more than once.

For most companion use cases (decision review, brainstorming, "what am I missing?"), no tools are needed — keep the call simple.

### When `claude -p` exits 1 with no output

This is the most confusing failure mode — Claude printed nothing, the exit code is 1. There are **two distinct causes**, and they need different fixes. Don't conflate them:

**1. Tool-permission missing (Claude refused to use a tool).**
The prompt asked Claude to do something action-y (run a command, read a file) but the tool wasn't on the allowlist. Fix: add `--allowedTools` with the specific tool/command, or rephrase as a text-only consultation.

**2. Filesystem/sandbox blocking Claude's state writes.**
Claude tried to write to its own state directory (e.g. `~/.claude/session-env/`, `~/.claude/projects/`) but the OS denied it with `EPERM`/`EACCES`. This happens when the *calling agent* (you, Codex, etc.) runs inside a sandbox that doesn't permit writes to `~/.claude/`. Fix: this is **your** sandbox, not Claude's — either escalate per-command, add `$HOME/.claude/` to your allow-write list, or relax your sandbox policy. Re-running with more flags will not help.

**How to tell which one it is:** run the same command outside `-p` (interactively) or with `--debug` and inspect the actual error. If you see `EPERM`/`EACCES` on a path under `~/.claude/`, it's #2. Otherwise it's #1.

### If `claude` is not installed

Tell the user: "I'd like to consult Claude on this but the `claude` CLI isn't available. Want to install it or should I proceed on my own?"

---

## Reporting back to the user

**After every Claude consultation**, surface the result to the user in 2–5 lines before you act. Format:

> **Consulted Claude on:** `<one-line topic>`
> **Claude's take:** `<core suggestion, 1–2 lines>`
> **Reason:** `<why, 1 line>`
> **Risk / caveat:** `<if any>`
> **What I'm going to do:** `<your action>`

The user must see this so they can intervene. Never silently follow Claude's advice — they explicitly asked for visibility into these consultations.

If Claude's answer conflicts with something the user already said: **the user wins.** Surface the conflict explicitly: "Claude suggested X but you earlier said Y — which way do you want to go?"

---

## Good prompts vs bad prompts

**Bad** (no context, Claude has to guess):
```
"Should I use Redis or Postgres for this?"
```

**Good** (Claude can actually help):
```
[CONTEXT]
Project: real-time leaderboard for a small game (~5k DAU).
Stack: Node 20, currently using Postgres for everything.
What I've done so far: built the scoring API, writing scores ~10/sec at peak.
Constraint: solo developer, wants minimal ops overhead.

[QUESTION]
Should I add Redis for the leaderboard reads, or is Postgres + a materialized
view enough at this scale?

[WHAT WOULD BE A GOOD ANSWER]
Recommendation + the scale threshold at which I should reconsider.
```

---

## What NOT to delegate to Claude

Claude is a peer, not an oracle. The **user** is still the decision-maker for:
- Anything destructive (`rm -rf`, `git push --force`, dropping tables, deleting branches).
- Business / product calls.
- Anything that depends on the user's private context (their team, their goals, their preferences).
- Final approval before merging, deploying, or sending anything external.

For those: ask the **user**, not Claude.

---

## Quick reference

| Situation                              | Action                                                          |
| -------------------------------------- | --------------------------------------------------------------- |
| Complex decision, multiple paths       | Consult Claude with both options + tradeoffs question.          |
| Stuck after 2 attempts                 | Consult Claude with what you tried and the error.               |
| Reviewing your own plan before acting  | Consult Claude with the plan and ask "what am I missing?"       |
| Destructive / irreversible action      | Ask the **user**, not Claude.                                   |
| Product / business / personal decision | Ask the **user**, not Claude.                                   |
| Trivial / obvious                      | Just do it, no consultation needed.                             |
