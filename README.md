# codex-review

A [Claude Code](https://claude.com/claude-code) skill that delegates a coding task to the [Codex CLI](https://github.com/openai/codex) and has Claude act as the reviewer.

Codex proposes each step read-only, Claude reviews and approves it (asking you only for risky actions such as deletes, installs, network access or git pushes), Codex applies it, and Claude verifies the diff, runs tests and commits a git checkpoint per step.

## Install

```sh
git clone https://github.com/ILKAY-BRAHIM/codex-review ~/.claude/skills/codex-review
```

Requires the Codex CLI (`npm install -g @openai/codex`) and `codex login`.

## Use

In Claude Code, say "let codex do it" / "use codex", or run:

```
/codex-review <task> [project directory]
```
