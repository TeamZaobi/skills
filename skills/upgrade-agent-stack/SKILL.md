---
name: upgrade-agent-stack
description: Use when the user wants to update locally installed AI terminal tools or git-backed skill repos in a scoped way, especially tools like Claude Code, Codex, Gemini CLI, Kimi, kimicc, or external skills. This skill helps inspect install methods, upgrade only named targets, skip dirty repos, and verify versions afterward.
---

# Upgrade Agent Stack

This skill is for updating a local AI terminal stack in a bounded way.

It is not for whole-machine package maintenance.

## Rules

1. Only touch tools or repos the user named, or a clearly bounded set such as "external skills" or "common AI CLIs".
2. Do not run blanket `brew upgrade` or `npm -g update`.
3. For npm-backed CLIs, update explicit packages by name.
4. For git repos, skip dirty worktrees.
5. If a repo is ahead of `origin`, report it instead of pulling.
6. If a repo is diverged or not fast-forwardable, stop and explain.
7. Verify versions after each update batch.

## Workflow

### 1. Scope The Request

Classify the target as:

- CLI tools
- skill repos
- both

Keep the scope on AI terminal tooling even if the user says "update everything".

### 2. Inspect Install Method First

Use shell inspection before upgrading anything:

```bash
which -a claude gemini kimicc codex kimi
npm -g ls --depth=0 2>/dev/null | rg -i 'claude|gemini|kimi|kimicc|codex'
uv tool list 2>/dev/null | rg -n 'kimi|kimi-cli'
brew list --formula --versions 2>/dev/null | rg -i 'claude|gemini|kimi|kimicc|codex'
```

For git-backed skill repos, inspect:

```bash
git status --short --branch
git rev-list --left-right --count HEAD...@{upstream}
git remote -v
```

### 3. Update By Type

- npm global package:
  - `npm outdated -g <pkg>`
  - `npm install -g <pkg>@latest`
- `uv tool`:
  - `uv tool upgrade <tool>`
- native updater:
  - prefer the tool's own update command, for example `claude update`
- Homebrew-linked Node CLI:
  - if the binary resolves into `lib/node_modules`, follow the underlying npm package rather than treating it as a brew formula

Common mappings:

- `gemini` -> `@google/gemini-cli`
- `codex` -> `@openai/codex`
- `kimicc` -> `kimicc`
- `kimi` binary usually comes from `uv tool` package `kimi-cli`

### 4. Update Skill Repos Safely

Only pull when the repo is clean and behind upstream:

```bash
git fetch --prune origin
git pull --ff-only
```

Default rules:

- dirty: skip
- ahead only: report, do not pull
- behind only and clean: fast-forward pull
- diverged: stop and explain

If the user says "external skills only", use the GitHub owner from `origin` and exclude the user's own owner.

### 5. Verify

Rerun the relevant version commands and summarize:

- updated
- already current
- skipped
- blocked

## Notes

- Prefer `claude update` first for Claude Code.
- If Claude Code fails on updater transport or certificate checks, verify the official latest-version endpoint first, then use the tool's documented fallback only if needed.
- Do not use `kimicc --version` as the primary check if it launches Claude Code instead of printing a plain version.
