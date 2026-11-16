@AGENTS.md

# CLAUDE.md

## Where guidance lives

- Shared rules for all agents live in `AGENTS.md`, imported above. That covers code style, tests,
  pre-commit checks, git rules, environment files and the docs links. Edit those rules there, not
  here.
- Repo layout and app structure: see the "Repo layout" section of `AGENTS.md`.
- This file holds only what is specific to Claude Code: its skills, subagents, hooks and MCP
  servers, plus Claude-only gotchas. Do not duplicate `AGENTS.md` content here.
- Claude Code configuration:
  - Skill: `.claude/skills/ci/SKILL.md`
  - Subagent: `.claude/agents/security-reviewer.md`
  - Hooks and MCP server selection: `.claude/settings.local.json`
  - MCP server definitions: `.mcp.json`

## Claude Code commands and gotchas

- **`/ci` is user-only.** The skill sets `disable-model-invocation: true`, so only the user can
  start it. Ask the user to run it and do not try to invoke it yourself.
- **`security-reviewer` subagent.** Spawn it after changes to views, querysets, permissions,
  middleware or auth code.
- **Ruff hook on Python edits.** A PostToolUse hook (see `.claude/settings.local.json` for the exact
  command and timeout) runs `ruff check --fix` on every `*.py` file after an Edit or Write. The hook
  may rewrite the file, so re-read it before editing it again. The hook does not format, so still
  run `uv run ruff format .`.
- **MCP servers** are defined in `.mcp.json`; both need `SONARQUBE_TOKEN` and `SONARQUBE_ORG` in the
  environment. `sonarqube` (Docker stdio) is the enabled default; `sonarqubeCloud` (HTTP) is the
  fallback. Findings reflect the last scanner upload, not the working tree, and
  `mcp__sonarqube__analyze_code_snippet` checks a single file.
