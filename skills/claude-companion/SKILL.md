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
