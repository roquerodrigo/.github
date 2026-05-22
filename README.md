# roquerodrigo/.github

Reusable GitHub Actions workflows shared across all my Home Assistant
custom integrations and Python SDKs.

## Why

Before this repo, every integration carried its own copy of the same
`lint.yml`, `tests.yml`, `validate.yml`, etc. Workflow fixes meant a PR
in every repo. Drift was inevitable.

Now each caller has **one `ci.yml`** that wires up jobs from this repo.
Update once here, tag a new version, every caller picks it up on their
next pin bump.

## Layout

```
.github/workflows/
├── ha-lint.yml            # ruff + mypy on custom_components/<package>
├── ha-tests.yml           # pytest + coverage (default 95 %)
├── ha-validate.yml        # hassfest + HACS
├── ha-codeql.yml          # CodeQL (security-extended)
├── ha-release.yml         # release-please
├── ha-auto-assign.yml     # add @roquerodrigo on new PR
├── ha-update-pr-branch.yml# auto-rebase PR onto main
│
├── sdk-lint.yml           # uv + ruff + mypy
├── sdk-tests.yml          # uv + pytest --cov (default 80 %)
├── sdk-codeql.yml         # CodeQL
└── sdk-release.yml        # release-please + PyPI publish via OIDC
```

## Caller pattern

Each consuming repo has a single `.github/workflows/ci.yml` that wires
the jobs with the right triggers, conditions and dependencies. See
[`ha-integration-blueprint`](https://github.com/roquerodrigo/ha-integration-blueprint/blob/main/.github/workflows/ci.yml)
for the canonical HA caller and
[`neakasa-litterbox-sdk`](https://github.com/roquerodrigo/neakasa-litterbox-sdk/blob/main/.github/workflows/ci.yml)
for the canonical SDK caller.

### Full HA caller

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 0 * * 0"   # CodeQL weekly

permissions:
  contents: read

jobs:
  lint:
    if: github.event_name != 'schedule'
    uses: roquerodrigo/.github/.github/workflows/ha-lint.yml@v2

  tests:
    if: github.event_name != 'schedule'
    needs: lint
    uses: roquerodrigo/.github/.github/workflows/ha-tests.yml@v2

  validate:
    if: github.event_name != 'schedule'
    needs: lint
    uses: roquerodrigo/.github/.github/workflows/ha-validate.yml@v2

  codeql:
    permissions:
      actions: read
      contents: read
      security-events: write
    uses: roquerodrigo/.github/.github/workflows/ha-codeql.yml@v2

  auto-assign:
    if: github.event_name == 'pull_request' && github.event.action == 'opened'
    permissions:
      pull-requests: write
    uses: roquerodrigo/.github/.github/workflows/ha-auto-assign.yml@v2
    secrets: inherit

  update-pr-branch:
    if: github.event_name == 'pull_request'
    needs: [lint, tests, validate]
    permissions:
      contents: write
      pull-requests: write
    uses: roquerodrigo/.github/.github/workflows/ha-update-pr-branch.yml@v2

  release:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: [lint, tests, validate]
    permissions:
      contents: write
      pull-requests: write
    uses: roquerodrigo/.github/.github/workflows/ha-release.yml@v2
    secrets: inherit
```

## Permissions model

The workflow-level `permissions: contents: read` is the baseline. Jobs
that call reusables needing more declare an explicit `permissions:`
block — GitHub refuses to start the workflow if a reusable requests
more than the caller allows. Jobs without explicit permissions inherit
the workflow-level baseline.

## Versioning

Tags follow `vN` (major only). Callers pin `@vN` and pick up
non-breaking changes automatically. A breaking change means a new
`vN+1` tag, and callers move at their own pace.

### `v2` — HA reusables migrated to uv (2026-05-22)

`ha-lint.yml` and `ha-tests.yml` were updated to install dependencies
via `uv sync --group dev [--group lint]` instead of
`pip install -r requirements.txt`. Callers on `@v2` need:

- `pyproject.toml` with `[dependency-groups]` `dev` (and `lint` for
  the lint reusable). The old `requirements.txt` /
  `requirements_test.txt` files are no longer used.
- `uv.lock` committed.
- `[tool.uv] package = false` for HA integrations (which aren't
  pip-installable packages — they're `custom_components/<name>/`
  directories).
- The `package` input on `ha-lint.yml` is gone; configure mypy's
  target via `[tool.mypy] files = ["custom_components/<name>"]` in
  the consumer's `pyproject.toml`.

`@v1` stays pinned at the pip-based commit for repos that haven't
migrated yet.

The SDK reusables (`sdk-*.yml`) were already uv-based and are
unchanged between `v1` and `v2`.

## Coverage gates

Coverage thresholds live in the consumer's `pyproject.toml`
(`[tool.pytest.ini_options]` `addopts = ["--cov-fail-under=N"]`) so
local runs and CI agree. Suggested defaults: **95 %** for HA
integrations, **80 %** for SDKs.

## Conventions

- Third-party actions pinned by full SHA; first-party (`googleapis`,
  `pypa`) pinned by major tag.
- `permissions: {}` at workflow scope when possible; jobs declare only
  what they need.
- All reusable workflows are read-only on `contents:` except `release`
  and `auto-assign`/`update-pr-branch` which need write.
