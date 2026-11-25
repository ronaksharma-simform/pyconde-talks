@AGENTS.md

# CLAUDE.md

## Where guidance lives

- Shared rules (code style, tests, pre-commit checks, git rules, environment files, docs links, repo
  layout) live in `AGENTS.md`, imported above. Edit them there, not here.
- This file holds only what is specific to Claude Code. Its configuration:
  - Skill: `.claude/skills/ci/SKILL.md`
  - Subagent: `.claude/agents/security-reviewer.md`
  - Hooks and MCP server selection: `.claude/settings.local.json`
  - MCP server definitions: `.mcp.json`
- If a path above no longer exists, update this file in the same change.

## Claude Code commands and gotchas

- **`/ci` is user-only.** The skill sets `disable-model-invocation: true`, so only the user can
  start it. Ask the user to run it and do not try to invoke it yourself.
- **`security-reviewer` subagent.** Spawn it after changes to views, querysets, permissions,
  middleware or auth code.
- **Ruff hook on Python edits.** A PostToolUse hook (matcher `Edit|Write`, timeout 30) runs
  `uv run ruff check --fix "<file>"` on every `*.py` file after an Edit or Write. The hook may
  rewrite the file, so re-read it before editing it again. The hook does not format, so still run
  `uv run ruff format .`. If the hook fails, run that command by hand and tell the user.
- **MCP servers** are defined in `.mcp.json`; both need `SONARQUBE_TOKEN` and `SONARQUBE_ORG` in the
  environment. `sonarqube` (Docker stdio) is the enabled default; `sonarqubeCloud` (HTTP) is the
  fallback. Findings reflect the last scanner upload, not the working tree, and
  `mcp__sonarqube__analyze_code_snippet` checks a single file. If `SONARQUBE_TOKEN` is unset, skip
  SonarQube checks and tell the user instead of retrying.

- Config test branch: CONFIG-TEST-V1
