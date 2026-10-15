---
name: codex-review
description: Delegate a coding task to the Codex CLI and act as its reviewer, step by step. Codex proposes each step read-only, Claude reviews and approves it (stopping to ask the user only for risky actions), Codex applies it, Claude verifies the diff and checkpoints with git. Use when the user says "use codex", "let codex do it", "codex as worker / you as reviewer", or invokes /codex-review.
---

# Codex as worker, Claude as reviewer

Codex does the work. You review every step before and after it happens. You approve normal steps yourself and **stop and ask the user only for risky actions**.

Arguments: the task description, optionally a project directory. If either is missing, ask once.

## Setup

1. Resolve the project directory (`$DIR`, absolute). Run every codex command from inside it (`cd "$DIR" && codex ...`) because `codex exec resume` has no `--cd` flag and filters sessions by cwd.
2. Make sure it is a git repo. If not, ask the user before running `git init`. If the working tree is dirty, tell the user and ask whether to commit or stash first, so Codex's changes can be diffed cleanly.
3. Record the starting point: `BASE=$(git rev-parse HEAD)` (or note "no commits yet").
4. Keep Codex output in the scratchpad: `OUT=<scratchpad>/codex`, `mkdir -p $OUT`.

## Model and effort selection (you decide, per call)

Always pass `-m "$MODEL"` and `-c model_reasoning_effort="\"$EFFORT\""` explicitly on every codex call — the user's `~/.codex/config.toml` default may be an expensive model at maximum effort, and you, not the config, choose.

Read the live model list first (models change; don't rely on memory):

```bash
python3 -I -c "import json;d=json.load(open('$HOME/.codex/models_cache.json'))
for m in d['models']: print(m['slug'],'|',[x['effort'] for x in m.get('supported_reasoning_levels',[])],'|',m.get('description',''))"
```

Map the models into three tiers by their descriptions: **fast** ("fast and affordable", e.g. a *luna* model), **workhorse** ("latest workhorse", e.g. the newest *sol*), **frontier** ("frontier intelligence", e.g. *astra*). Prefer the newest generation in each tier; skip legacy/older and special-purpose models (e.g. `codex-auto-review`). Only use an effort the chosen model lists as supported.

| Call | Default choice |
|---|---|
| Initial plan | frontier, `high` |
| Trivial step (rename, typo, docs, config value, boilerplate) | fast, `low`–`medium` |
| Normal step (feature code, refactor of one module, tests) | workhorse, `medium` |
| Hard step (tricky algorithms, concurrency, cross-module refactor, security/auth logic, debugging a non-obvious failure) | frontier, `high` |
| Re-propose after your feedback | same model, same or +1 effort |

**Escalation:** if a step's proposal is rejected twice, or tests still fail after a fix attempt, raise effort one level; if already `high`, move up a tier. Use `xhigh` only after escalation; `max`/`ultra` only for a genuinely stuck hard step, and tell the user when you do (they are slow and costly). **De-escalate** back to the default for the next step.

State the choice in each step report, e.g. `step 3 · gpt-6.1-sol/medium · approved · tests pass`. If the user names a model or effort, that overrides this policy for the rest of the task.

## Start the session (read-only plan)

```bash
cd "$DIR" && codex exec -m "$MODEL" -c model_reasoning_effort="\"$EFFORT\"" --sandbox read-only --json -o "$OUT/plan.md" "<prompt>" > "$OUT/plan.jsonl"
```

Prompt template:

> Task: <task>. Do NOT modify anything yet. Explore the code, then reply with a numbered plan of small, independently verifiable steps. For each step list: files to change, what changes, and any shell commands you intend to run (installs, migrations, deletes, network access). Then stop.

Get the session id from the first JSON event (`thread.started` → `thread_id`):

```bash
SID=$(grep -m1 -o '"thread_id":"[^"]*"' "$OUT/plan.jsonl" | cut -d'"' -f4)
```

If that is empty, fall back to `--last` (still run from `$DIR`). If codex fails with an auth error, tell the user to run `codex login` themselves and stop.

Review the plan: does it actually solve the task, are steps small, is anything missing or unnecessary? If not acceptable, resume with feedback (read-only) until it is. Briefly show the user the accepted plan.

## Per-step loop

For each step N:

**1. Propose (read-only)**
```bash
cd "$DIR" && codex exec resume "$SID" -m "$MODEL" -c model_reasoning_effort="\"$EFFORT\"" -c sandbox_mode='"read-only"' -o "$OUT/step$N-proposal.md" \
  "Propose step $N only: exact code changes (as a diff or precise description) and any commands. Do not apply anything."
```

**2. Review the proposal.** Check correctness, edge cases, fit with existing code style, scope creep, and risk (see below).
- Problems → resume with concrete feedback, re-propose. Max 3 rounds per step; then ask the user.
- Risky → STOP and ask the user (AskUserQuestion) with a short summary of what and why. Proceed only on approval.
- Otherwise → approve yourself and say so in one line.

**3. Apply (workspace-write)**
```bash
cd "$DIR" && codex exec resume "$SID" -m "$MODEL" -c model_reasoning_effort="\"$EFFORT\"" -c sandbox_mode='"workspace-write"' -o "$OUT/step$N-apply.md" \
  "Approved. Apply step $N exactly as proposed. Do not do anything beyond this step. Summarize what you did."
```
Include any adjustments you required in the approval message.

**4. Verify the result yourself** — never trust Codex's summary alone:
- `git status` and `git diff` — changes must match the approved proposal. Unexpected files or edits → revert them (`git checkout -- <file>` / remove new files after checking) and redo the step, or ask the user if unclear.
- Read the changed code; look for bugs.
- Run the relevant tests / linter / build if the project has them. Failures → send output back to Codex to fix (same propose→review→apply cycle).

**5. Checkpoint.** `git add -A && git commit -m "codex step $N: <summary>"` (with the commit attribution lines from the system reminder). This makes every step reversible.

Report each step to the user in 1–2 lines: what changed, review verdict, test result.

## Risky actions — always stop and ask the user

- Deleting files or directories, or large rewrites of existing files
- Anything outside `$DIR`, or touching config/dotfiles, credentials, `.env`, secrets
- Installing/removing dependencies, changing lockfiles substantially
- Network access, downloads, `curl | sh`, calls to external services or APIs
- Database migrations, data deletion, schema changes
- git history operations (push, force-push, rebase, reset of user commits, branch deletion)
- Deploys, publishing, sending anything externally
- Changes to auth, permissions, security, or payment logic
- Anything needing more than `workspace-write` sandbox — never use `danger-full-access` or `--dangerously-bypass-approvals-and-sandbox`

When in doubt, treat it as risky.

## Finish

1. Final review of the whole change: `git diff $BASE..HEAD` (or full log if no base). Check the task is fully done and nothing unrelated changed.
2. Run the full test suite if available.
3. Optionally run `codex exec review` as a second opinion, but your own review is the authority.
4. Summarize to the user: steps done, files changed, issues found and fixed during review, test results, anything left open, and that each step is a separate commit they can revert. Do not push unless the user asks.
