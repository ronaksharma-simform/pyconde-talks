---
name: ci
description: >-
  Run the local CI pipeline and refresh SonarQube: ruff, bandit and pytest with JSON and coverage
  reports, a coverage.xml guard, a sonar-scanner upload, then fetch the new SonarQube issues. Use
  when the user asks to run CI, do a pre-PR check, lint, run the security scan and tests with
  coverage, refresh the SonarCloud dashboard, or when SonarQube findings look stale.
disable-model-invocation: true
allowed-tools: >-
  Bash(uv run:*), Bash(uv sync:*), Bash(sonar-scanner:*), Bash(rm:*), Bash(test:*), Bash(grep:*),
  Bash(tee:*), mcp__sonarqube__search_sonar_issues_in_projects
---

# /ci: full local CI + SonarQube refresh

Run this before opening a PR, or whenever the `mcp__sonarqube__*` findings look stale. The scanner
uploads the JSON reports that ruff, bandit, and pytest produce, so those have to run first.

Before you start:

- `ruff check --fix` rewrites files in place across the whole repo. Commit or stash first, so the
  changes can be reviewed or reverted with `git diff` / `git checkout`.
- The scan uploads to the shared SonarCloud project. `SONAR_TOKEN` must come from the environment;
  never put it in a file or on the command line.

## Steps

1. Drop stale reports so the scanner never uploads yesterday's data.
2. Sync dev and test deps.
3. Run ruff with a JSON report (exit-zero so the chain continues even if there are findings).
4. Run bandit with a JSON report (exit-zero, same reason).
5. Run the full pytest suite (must pass; pytest-cov writes `reports/coverage.xml`).
6. Assert `reports/coverage.xml` exists and is non-empty. This guards against the "0% coverage" trap
   where `sonar-scanner` silently ships an empty report.
7. Invoke `sonar-scanner`, tee the log to `reports/sonar-scan.log`.
8. Assert the log reports `ANALYSIS SUCCESSFUL`.

```bash
set -o pipefail; \
rm -f reports/coverage.xml reports/lcov.info reports/ruff_report.json reports/bandit_report.json && \
  uv sync --group dev --group test && \
  uv run ruff check --fix --output-format=json --output-file=reports/ruff_report.json --exit-zero . && \
  uv run bandit --configfile pyproject.toml --recursive --format json --output reports/bandit_report.json --exit-zero . && \
  uv run pytest && \
  test -s reports/coverage.xml && \
  sonar-scanner 2>&1 | tee reports/sonar-scan.log && \
  grep -q 'ANALYSIS SUCCESSFUL' reports/sonar-scan.log
```

After the scanner finishes, use `mcp__sonarqube__search_sonar_issues_in_projects` to pull new
findings for review.

## Example

- Success: pytest passes, `reports/coverage.xml` is non-empty, the log ends with
  `ANALYSIS SUCCESSFUL`, then the MCP tool lists the new issues.
- Failure: two tests fail, so the chain stops at step 5 and nothing is uploaded. Fix the tests and
  rerun the whole chain.

## Notes

- pytest is the gate: if tests fail, the chain stops before the scanner runs.
- The `test -s reports/coverage.xml` guard is the second gate. If coverage.xml is missing or empty
  the chain stops before upload, so the SonarCloud dashboard never flips back to 0%.
- `set -o pipefail` makes a failing `sonar-scanner` fail the chain instead of being masked by `tee`.
- ruff and bandit are set to `--exit-zero` on purpose so report JSON is always written.
- If a gate fails: show the failing step's output, fix the cause (do not bypass it), and rerun the
  whole chain from the start. If `sonar-scanner` fails on authentication, check `SONAR_TOKEN`.
- Do **not** skip this pipeline to make findings disappear. If something is flaky, fix the flake.
- Do **not** run `sonar-scanner` on its own. Without a fresh `reports/coverage.xml` it uploads an
  analysis with 0% coverage and overwrites the last good one.
