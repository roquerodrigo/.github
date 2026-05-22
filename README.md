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

### Minimum HA caller

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule:
    - cron: "0 0 * * 0"

permissions: {}

jobs:
  lint:
    if: github.event_name != 'schedule'
    uses: roquerodrigo/.github/.github/workflows/ha-lint.yml@v1
    with: { package: my_integration }

  tests:
    if: github.event_name != 'schedule'
    needs: lint
    uses: roquerodrigo/.github/.github/workflows/ha-tests.yml@v1
    with: { package: my_integration }

  validate:
    if: github.event_name != 'schedule'
    needs: lint
    uses: roquerodrigo/.github/.github/workflows/ha-validate.yml@v1

  codeql:
    uses: roquerodrigo/.github/.github/workflows/ha-codeql.yml@v1

  release:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: [lint, tests, validate]
    uses: roquerodrigo/.github/.github/workflows/ha-release.yml@v1
    secrets: inherit
```

Add `auto-assign` and `update-pr-branch` jobs guarded by
`if: github.event_name == 'pull_request'` for the full pipeline.

## Versioning

Tags follow `vN` (major only). Callers pin `@v1` and pick up
non-breaking changes automatically. A breaking change means a `v2` tag,
and callers move at their own pace.

## Coverage gates

- HA integrations default to **95 %** (`ha-tests.yml`, override via
  `cov-fail-under`).
- SDKs default to **80 %** (`sdk-tests.yml`, override via
  `cov-fail-under`).

## Conventions

- Third-party actions pinned by full SHA; first-party (`googleapis`,
  `pypa`) pinned by major tag.
- `permissions: {}` at workflow scope when possible; jobs declare only
  what they need.
- All reusable workflows are read-only on `contents:` except `release`
  and `auto-assign`/`update-pr-branch` which need write.
