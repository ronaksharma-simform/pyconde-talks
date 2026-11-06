@AGENTS.md

# CLAUDE.md

## Where guidance lives

- Shared rules for all agents live in `AGENTS.md`, imported above. That covers code style, tests,
  pre-commit checks, git rules, environment files and the docs links. Edit those rules there, not
  here.
- This file holds only what is specific to Claude Code: its skills, subagents, hooks and MCP
  servers, plus Claude-only gotchas. Do not duplicate `AGENTS.md` content here.
- Claude Code configuration:
  - Skill: `.claude/skills/ci/SKILL.md`
  - Subagent: `.claude/agents/security-reviewer.md`
  - Hooks and MCP server selection: `.claude/settings.local.json`
  - MCP server definitions: `.mcp.json`

## Claude Code commands and gotchas

- **`/ci` is user-only.** The skill (`.claude/skills/ci/SKILL.md`) sets
  `disable-model-invocation: true`, so only the user can start it. Ask the user to run it and do not
  try to invoke it yourself. `AGENTS.md` says Claude "may" invoke it and links to
  `.claude/skills/ci.md`; both are wrong for this skill.
- **`security-reviewer` subagent.** Defined in `.claude/agents/security-reviewer.md`. Spawn it after
  changes to views, querysets, permissions, middleware or auth code.
- **Ruff hook on Python edits.** A PostToolUse hook in `.claude/settings.local.json` runs
  `uv run ruff check --fix <file>` after every Edit or Write of a `*.py` file (30s timeout). The
  hook may rewrite the file, so re-read it before editing it again. The hook does not format, so
  still run `uv run ruff format .`.
- **MCP servers** are defined in `.mcp.json`. Both need `SONARQUBE_TOKEN` and `SONARQUBE_ORG` in the
  environment.
  - `sonarqube` is a Docker stdio server and the enabled default.
  - `sonarqubeCloud` is an HTTP fallback.
  - Findings reflect the last scanner upload, not the working tree.
  - `mcp__sonarqube__analyze_code_snippet` checks a single file.
- **Quick commands** (the full list is in `AGENTS.md`):
  - `uv run pytest -k <name>`: run a single test.
  - `uv run ruff check --fix .` and `uv run ruff format .`: lint and format.
  - `zuban check`: strict type check.
  - `prek run -a`: run all pre-commit hooks.
  - `uv run zizmor .`: audit workflows.
