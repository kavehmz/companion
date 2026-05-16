---
name: codex-companion
description: Use when an AI agent working in a local project needs to consult a resumed Codex conversation as a second brain for complex decisions, uncertain implementation choices, stuck debugging, code/spec review, or high-risk changes. The skill gives a context-handoff protocol, Codex CLI resume workflow, human-decision boundaries, and required result reflection back to Kaveh.
---

# Codex Companion

Use Codex as a project companion when another AI is working on Kaveh's local project and needs a second opinion, decision review, debugging help, or a bounded implementation assist. Codex is a peer reviewer and collaborator, not a replacement for Kaveh on product, policy, legal, spend, hiring, security, or other human-critical decisions.

## Use Criteria

Call Codex when one of these is true:

- The current decision has multiple plausible technical paths and the wrong choice would be costly.
- You are stuck after a real investigation and need another agent to inspect evidence or propose next moves.
- You are about to make a cross-cutting implementation, spec, prompt, migration, or architecture change.
- You need a code-backed review of a plan, diff, failing test, runtime error, or contract drift.
- You need help turning messy evidence into a concise recommendation for Kaveh.

Do not call Codex for trivial edits, simple commands, or decisions that obviously require Kaveh's direct judgment. For human-critical decisions, ask Codex to sharpen options and risks, then bring the decision back to Kaveh.

## Context Handoff

Always provide fresh context. Resuming a Codex session restores prior conversation history, but Codex will not know what happened inside your separate AI session unless you tell it.

Send a structured prompt with these fields:

```markdown
Project: <repo/workspace path and branch if known>
User request: <what Kaveh asked for>
Current state: <files changed, commands run, tests, errors, relevant outputs>
What I tried: <short factual list>
Uncertainty: <the exact question, tradeoff, or blocker>
Options I see: <option A/B/C with concrete consequences, if known>
Human boundary: <what must go back to Kaveh instead of being decided by agents>
Ask for Codex: <review plan, debug, inspect files, propose patch, implement bounded change, etc.>
Desired output: <decision recommendation, patch summary, test command, user-facing reply, etc.>
```

If asking Codex to edit files, define a narrow ownership scope and avoid simultaneous edits to the same files. After Codex returns, inspect the resulting diff or advice before acting on it.

## CLI Workflow

For the normal case, call Codex from the project directory and resume the latest session for that cwd. Use the non-interactive form so the other AI can pass a full context handoff:

```bash
cd /path/to/current/project
/Applications/Codex.app/Contents/Resources/codex exec resume --last - <<'PROMPT'
Project: ...
User request: ...
Current state: ...
What I tried: ...
Uncertainty: ...
Options I see: ...
Human boundary: ...
Ask for Codex: ...
Desired output: ...
PROMPT
```

If `codex` is on `PATH`, this shorter form is equivalent:

```bash
codex exec resume --last - <<'PROMPT'
...
PROMPT
```

Use the interactive form only when a human is driving the terminal:

```bash
codex resume --last
```

Use an explicit session id when Kaveh provides one, or when the latest session for the directory is not the intended companion conversation:

```bash
codex exec resume <session-id> - <<'PROMPT'
...
PROMPT
```

Resolve the Codex binary in this order:

```bash
command -v codex
/Applications/Codex.app/Contents/Resources/codex
```

Persisted Codex sessions normally live under:

```text
${CODEX_HOME:-$HOME/.codex}/sessions/<date>/rollout-*.jsonl
```

Each `session_meta` record contains the session id, timestamp, cwd, and source file path. Normally you do not need this metadata; use it only to disambiguate multiple sessions.

## Result Reflection

After consulting Codex, briefly tell Kaveh what happened. Do not hide the second-agent conversation.

Use this shape in your next user-facing update:

```markdown
I consulted Codex on <question>. Codex recommended <decision/next step> because <main reason>. I applied/kept <actions taken or not taken>. Remaining human decision: <only if needed>.
```

If Codex changed files, include the files changed and the verification run. If Codex only advised, say that no files were changed by Codex.

## Guardrails

- Treat Codex output as advice until verified against the project.
- Do not let Codex make irreversible or external side-effect decisions without Kaveh.
- Do not paste secrets, private tokens, or unnecessary personal data into the prompt.
- Prefer concise evidence over raw logs; include exact error text and file paths when they matter.
- If no matching Codex session is found, ask Kaveh for the right session id or permission to start a new Codex session for this project.
