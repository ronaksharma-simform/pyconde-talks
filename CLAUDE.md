@AGENTS.md

# CLAUDE.md

## Where guidance lives

- Read `AGENTS.md` (imported above) for repo layout, test conventions and git rules before making
  code changes. It also covers code style, pre-commit checks, environment files and the docs links.
  Edit those rules there, not here.
- This file holds only what is specific to Claude Code: its skills, subagents, hooks and MCP
  servers, plus Claude-only gotchas. Do not duplicate `AGENTS.md` content here.

## Claude Code config and gotchas

- **`/ci` is user-only.** The skill (`.claude/skills/ci/SKILL.md`) sets
  `disable-model-invocation: true`, so only the user can start it. Ask the user to run it and do not
  try to invoke it yourself.
- **`security-reviewer` subagent** (`.claude/agents/security-reviewer.md`). Spawn it after changes
  to views, querysets, permissions, middleware or auth code. Open its file only when changing that
  kind of code.
- **Ruff hook on Python edits** (`.claude/settings.local.json`, which also selects the enabled MCP
  servers; open it only when changing hooks or MCP selection). A PostToolUse hook runs
  `uv run ruff check --fix <file>` after every Edit or Write of a `*.py` file (30s timeout). The
  hook may rewrite the file, so re-read it before editing it again. The hook does not format, so
  still run `uv run ruff format .`.
- **MCP servers** are defined in `.mcp.json`; open it only when SonarQube tools fail to connect.
  Both need `SONARQUBE_TOKEN` and `SONARQUBE_ORG` in the environment.
  - `sonarqube` is a Docker stdio server and the enabled default.
  - `sonarqubeCloud` is an HTTP fallback.
  - Findings reflect the last scanner upload, not the working tree.
  - `mcp__sonarqube__analyze_code_snippet` checks a single file.
- **Quick commands** that the ruff hook affects (everything else is in `AGENTS.md`):
  - `uv run ruff check --fix .`: lint with auto-fix.
  - `uv run ruff format .`: format.
