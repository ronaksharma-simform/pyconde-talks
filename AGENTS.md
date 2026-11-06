# AGENTS.md

Guidance for AI coding agents (Claude Code, Codex, etc.) working in this repository. Claude-specific
configuration (skills, subagents, hooks, MCP selection) lives in `CLAUDE.md`.

Django 6 app for PyCon DE / PyData Berlin talks and schedules. Stack: Python 3.14, Tailwind v4,
HTMX, SQLite (dev) / PostgreSQL 18 (prod), `uv` for packages.

## Layout

- `event_talks/`, `events/`, `talks/`, `users/`, `utils/`: Django project and apps
- `templates/`: Django templates (HTMX partials included)
- `docs/`: Zensical site; `docker/`: Docker Compose and image files
- Run the dev server: `uv run python manage.py runserver 8000`

See [docs/architecture/index.md](docs/architecture/index.md) for the full app layout.

## Code style

- Python 3.14: use modern syntax. PEP 695 generics, `match`, structural pattern matching, and PEP
  758 `except` without parentheses (`except ValueError, TypeError:`) are all fine.
- 100-column width for code, comments, and docstrings. Fill the line: do not wrap at 80 or 88 out of
  habit. The `docformatter` hook enforces most docstrings, but not all of them, and nothing enforces
  comments. See the note below.
- Write comments and docstrings in plain, simple language. Explain the _why_, not the _what_.
- Never start a docstring with a bare lowercase function name: docformatter capitalizes the first
  word, so `event() returns ...` silently becomes `Event() returns ...`, which names something else.
  Qualify it (`PretalxClient.event() returns ...`) or wrap it in double backticks.
- A function whose body is only a docstring needs a trailing comment, or docformatter and
  ruff-format will fight over the blank line after it.
- Never use em-dash (`—`) or en-dash (`–`). Rephrase instead: a colon, a semicolon, parentheses, or
  two sentences. A spaced hyphen is not a substitute, it means something else. Hyphens are for
  compound words (`event-scoped`) and ranges (`09:00 - 17:30`).

### What docformatter does not catch

It skips any docstring description that contains a colon or a hyphenated aside (it reads the line as
a field list), so the 100-column width is on you for those, and for every comment. Check them by
eye. Do not use `--force-wrap`: it flattens real `Attributes:` sections.

## Tests and code quality

- Add or update tests whenever you change behavior. Target: 90%+ coverage.
- `uv run pytest` (full suite) or `uv run pytest -k test_name` (single test).
- Keep functions small and focused. If a function grows complex (deep nesting, many branches, hard
  to test), refactor before adding more to it. Extract helpers, split responsibilities.
- SonarQube has many false positives. Silence with `# NOSONAR` (and a one-line reason) only when the
  finding clearly does not apply. Default is to fix, not silence, and never silence a finding just
  to make CI pass.
- When you change views, querysets, permissions, middleware, or auth code, consider spawning the
  `security-reviewer` subagent for a second pass (cross-event leaks, auth bypasses, PII in logs).
  Optional, not mandatory.

### SonarQube and Django templates

Its HTML analyzer does not know Django syntax. Two rules for `templates/`:

- Do not write a tag literally inside `{# ... #}` or `{% comment %}`: the analyzer parses it as a
  real element. Name elements in prose (`a nav element`), use `{placeholder}` not `<placeholder>`,
  and keep a comment out of a `<button>` or `<a>`.
- A partial is analyzed as a standalone file, so anything the including template supplies is
  invisible (for example a control's label). Mark such findings false positive, with the including
  template named as the reason.

`mcp__sonarqube__analyze_code_snippet` runs the real analyzer on one file, so a suspected false
positive can be confirmed (and a fix checked) without waiting for a full `/ci` upload.

## Before committing

```bash
uv run ruff check --fix .   # Lint + auto-fix
uv run ruff format .        # Format
zuban check                 # Strict type check (mypy-compatible, with django-stubs)
prek run -a                 # Run all pre-commit hooks (drop-in pre-commit replacement)
```

Anything under `.github/workflows/` is audited by [`zizmor`](https://docs.zizmor.sh)
(`uv run zizmor .`), as a pre-commit hook and in `zizmor.yml`, which also uploads SARIF to the
Security tab. A workflow edit must keep it clean: actions pinned to a full commit SHA,
`persist-credentials: false` on every checkout, and the narrowest `permissions` per job.

The full local CI + SonarQube refresh pipeline lives in the `/ci` skill
([.claude/skills/ci/SKILL.md](.claude/skills/ci/SKILL.md)). In Claude Code it is `/ci` and is
user-invoked only (`disable-model-invocation: true`), so ask the user to run it. Use it sparingly:

- Each run takes ~1-2 minutes and uploads to SonarCloud, so treat it as a verification step, not a
  loop.
- Run it at most twice per round of fixes, to confirm the findings actually cleared. Do not re-run
  just because the `mcp__sonarqube__*` tools still show yesterday's data; wait until there is a real
  set of changes to verify.

The `mcp__sonarqube__*` tools query findings from the last upload.

Two MCP servers are configured in `.mcp.json` for all agents: `sonarqube` (self-hosted Docker, full
toolset) and `sonarqubeCloud` (cloud-native HTTP, smaller toolset). Default is `sonarqube`. Only
switch to `sonarqubeCloud` if the Docker-based server is unavailable (e.g., Docker not running).

## Git

- Make small, atomic commits. One logical change per commit.
- **Commit as you go.** After each discrete change, run the pre-commit checks and commit before
  moving to the next. Do not batch unrelated edits into one large commit at the end of a session.
- Write clear commit messages: short imperative subject, body describing the _why_.

## Things agents must never do

- **Never push to GitHub** unless the user explicitly asks in this turn. Creating local commits is
  fine; `git push` is not.
- **Never skip hooks** (`--no-verify`, `--no-gpg-sign`). If a hook fails, fix the cause.
- **gpg-agent restart is pre-authorized.** If a commit hangs, times out, or fails with a pinentry /
  gpg-agent error (common after the laptop wakes from sleep), the agent MUST run
  `gpgconf --kill gpg-agent` itself via the shell and retry the commit. Do not stop and wait for
  the user: this command is the standard fix, safe to run at any time, and the commit cannot
  complete until the agent is restarted.
- **Never commit real secrets.** If `django-vars.env` or `docker/.env` shows non-placeholder values
  in a staged diff, stop and flag it.
- **Never rewrite shared history** (`git rebase`, `git reset --hard`, force-push) on any branch that
  has been pushed, unless the user asks.

## Environment files

`django-vars.env` (local) and `docker/.env` (Docker Compose) must stay in sync. When you add,
rename, or remove a variable in one, update the other in the same commit.

## Context management

When compacting, preserve: the list of files modified in this session, any failing tests or open
SonarQube findings not yet resolved, and any pending todos. Drop large file dumps and exploratory
search output.

## Further reading

The `docs/` tree is a [Zensical](https://zensical.org/) site, published to GitHub Pages at
<https://pioneershub.github.io/pyconde-talks/> by `.github/workflows/docs.yml`. Load these sources
only when relevant; they are not auto-imported.

- Day-to-day commands, setup, CI: [docs/development/index.md](docs/development/index.md)
- CI/CD pipeline, GHCR images, server deploy: [docs/deployment/ci-cd.md](docs/deployment/ci-cd.md)
- App layout and integrations: [docs/architecture/index.md](docs/architecture/index.md)
- Data model reference: [docs/architecture/data-model.md](docs/architecture/data-model.md)
- Pretalx importer (modes, detect-and-review, image regen):
  [docs/reference/pretalx-sync.md](docs/reference/pretalx-sync.md)
- Management command reference:
  [docs/reference/management-commands.md](docs/reference/management-commands.md)
- Translations / i18n (makemessages, compilemessages, adding a language):
  [docs/development/translations.md](docs/development/translations.md)
- Env var reference: [docs/getting-started/configuration.md](docs/getting-started/configuration.md)
  and [django-vars.env](django-vars.env)
- Editing the docs site: [docs/development/documentation.md](docs/development/documentation.md)
