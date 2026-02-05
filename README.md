# CI/CD Components

Reusable CI/CD pipeline components for org-wide adoption. Plug these into any repo with minimal configuration.

Supports **Python backends** (using `uv`) and **TypeScript + Vue frontends** with automatic detection.

## Quick Start

In your repo, create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
    types: [opened, reopened, synchronize, closed]
  push:
    branches: [main, develop]

jobs:
  ci:
    uses: your-org/ci-cd-components/.github/workflows/ci.yml@main
    secrets: inherit
    with:
      app_name: my-app
      fly_org: your-org
```

For releases, create `.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: your-org/ci-cd-components/.github/workflows/release.yml@main
    secrets: inherit
    with:
      app_name: my-app
```

That's it. The reusable workflow handles lint, build, scan, test, and deploy with Graphite CI optimization and Doppler secrets.

## Architecture

```
ci-cd-components/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml              # Main reusable CI workflow (PR + push)
│   │   └── release.yml         # Reusable release workflow (production deploy)
│   └── actions/
│       ├── lint/               # Ruff, mypy (Python) / ESLint, Prettier (JS)
│       ├── build/              # uv build (Python) / npm build (JS)
│       ├── scan/               # Gitleaks, pip-audit, Bandit, Semgrep, Trivy
│       ├── test/               # pytest (Python) / Vitest, Playwright (JS)
│       └── deploy/             # Fly.io deploy with Doppler secrets
├── renovate.json5              # Shared renovate config (extend in repos)
├── ci-cd-pipeline-components.md # Reference documentation
└── README.md
```

## Language Support

All actions auto-detect project type and run appropriate tools:

| Language | Detection | Package Manager | Lint | Test |
|----------|-----------|-----------------|------|------|
| Python | `pyproject.toml` or `uv.lock` | `uv` | Ruff, mypy | pytest |
| Node.js | `package.json` | npm | ESLint, Prettier | Vitest, Playwright |

## Required Secrets (in consuming repo)

| Secret | Description |
|--------|-------------|
| `GRAPHITE_CI_OPTIMIZER_TOKEN` | Graphite CI optimizer token |
| `DOPPLER_TOKEN_DEV` | Doppler service token for dev environment |
| `DOPPLER_TOKEN_STG` | Doppler service token for staging environment |
| `DOPPLER_TOKEN_PRD` | Doppler service token for production environment |
| `FLY_API_TOKEN` | Fly.io deploy token |

## Workflows

### `ci.yml` - Main CI Pipeline

Runs on PRs and pushes. Handles the full pipeline with Graphite CI optimization.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app_name` | Yes | - | App name (used for Fly app naming) |
| `fly_org` | No | `personal` | Fly.io organization |
| `fly_region` | No | `iad` | Fly.io region |
| `doppler_project` | No | `app_name` | Doppler project name |
| `node_version` | No | `20` | Node.js version |
| `skip_e2e` | No | `false` | Skip E2E tests |
| `runner` | No | `blacksmith` | GitHub runner to use |

### `release.yml` - Release Pipeline

Runs on push to main. Uses release-please to create releases and deploys to production.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app_name` | Yes | - | App name |
| `fly_org` | No | `personal` | Fly.io organization |
| `fly_region` | No | `iad` | Fly.io region |
| `doppler_project` | No | `app_name` | Doppler project name |
| `runner` | No | `blacksmith` | GitHub runner to use |

## Composite Actions

Each action can be used independently if you need custom workflows:

```yaml
- uses: your-org/ci-cd-components/.github/actions/lint@main
  with:
    node_version: '20'
    working_directory: '.'

- uses: your-org/ci-cd-components/.github/actions/test@main
  with:
    test_type: 'e2e'
    base_url: 'https://my-app-preview.fly.dev'

- uses: your-org/ci-cd-components/.github/actions/deploy@main
  with:
    environment: stg
    app_name: my-app-staging
    doppler_token: ${{ secrets.DOPPLER_TOKEN_STG }}
    doppler_project: my-app
    fly_api_token: ${{ secrets.FLY_API_TOKEN }}
```

### Action Inputs

#### `lint`
| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `20` | Node.js version |
| `working_directory` | `.` | Working directory (monorepo support) |

#### `build`
| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `20` | Node.js version |
| `working_directory` | `.` | Working directory |

#### `scan`
| Input | Default | Description |
|-------|---------|-------------|
| `working_directory` | `.` | Working directory |
| `upload_sarif` | `true` | Upload SARIF to GitHub Security tab |

#### `test`
| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `20` | Node.js version |
| `working_directory` | `.` | Working directory |
| `test_type` | `unit` | Test type: `unit`, `e2e`, or `all` |
| `base_url` | - | Base URL for E2E tests |

#### `deploy`
| Input | Required | Description |
|-------|----------|-------------|
| `environment` | Yes | Environment: `dev`, `stg`, or `prd` |
| `app_name` | Yes | Fly.io app name |
| `config_file` | No | Fly config file (default: `fly.toml`) |
| `doppler_token` | Yes | Doppler service token |
| `doppler_project` | Yes | Doppler project name |
| `fly_api_token` | Yes | Fly.io API token |

## Environments

The pipeline supports 4 environments:

| Environment | Trigger | Fly App | Doppler Config |
|-------------|---------|---------|----------------|
| Preview | PR opened | `<app>-pr-<number>` | - |
| Dev | Push to `develop` | `<app>-dev` | `dev` |
| Staging | Push to `main` | `<app>-staging` | `stg` |
| Production | Release created | `<app>` | `prd` |

## Graphite CI Optimization

The pipeline uses Graphite's CI optimizer to skip redundant runs on mid-stack PRs:

- **Bottom of stack:** Full CI (lint, build, scan, test, deploy preview)
- **Top of stack:** Full CI
- **Mid-stack PRs:** Lint only (fast, cheap)

Configure in Graphite dashboard per repo.

## Renovate Configuration

Extend the shared Renovate config in your repo's `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>your-org/ci-cd-components"]
}
```

Features:
- Automerge minor/patch updates
- Group updates by type (Python, Node, GitHub Actions)
- Security updates prioritized
- Weekly lock file maintenance
- `uv lock` post-upgrade for Python projects

## Customization

### Override specific steps

If you need to customize a step, use the composite actions directly:

```yaml
jobs:
  custom_ci:
    runs-on: blacksmith
    steps:
      - uses: actions/checkout@v4

      # Use shared lint
      - uses: your-org/ci-cd-components/.github/actions/lint@main

      # Custom build step
      - run: ./custom-build.sh

      # Use shared deploy
      - uses: your-org/ci-cd-components/.github/actions/deploy@main
        with:
          environment: stg
          app_name: my-app-staging
          doppler_token: ${{ secrets.DOPPLER_TOKEN_STG }}
          doppler_project: my-app
          fly_api_token: ${{ secrets.FLY_API_TOKEN }}
```

### Monorepo support

All actions support a `working_directory` input for monorepos:

```yaml
jobs:
  ci:
    uses: your-org/ci-cd-components/.github/workflows/ci.yml@main
    secrets: inherit
    with:
      app_name: my-backend
      working_directory: packages/backend
```

## Required Files in Consuming Repo

For full functionality, ensure your repo has:

**Python projects:**
- `pyproject.toml` with dev dependencies including `ruff`, `mypy`, `pytest`

**Node.js projects:**
- `package.json` with scripts: `lint`, `format:check`, `typecheck`, `test:unit`, `test:e2e`
- `package-lock.json`

**Fly.io deployment:**
- `fly.toml` (production)
- `fly.staging.toml` (staging)
- `fly.dev.toml` (dev)
- `fly.preview.toml` (PR previews)

## Reference

See [ci-cd-pipeline-components.md](./ci-cd-pipeline-components.md) for detailed documentation on each component, tools, and patterns.
