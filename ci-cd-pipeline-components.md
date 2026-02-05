# Reusable CI/CD Pipeline Components

General-purpose, reusable CI/CD pipeline components for org-wide adoption. Designed around a Fly.io deployment model with environment promotion (fleeting PR previews, dev, staging, production). PRs are managed via Graphite (stacked PRs) with CI optimization to control costs. Secrets are managed via Doppler.

---

## Table of Contents

- [Environments](#environments)
- [PR Management (Graphite)](#pr-management-graphite)
- [1. Lint](#1-lint)
- [2. Build](#2-build)
- [3. Scan](#3-scan)
- [4. Test](#4-test)
- [5. Deploy](#5-deploy)
- [6. Release](#6-release)
- [7. Renovate / Dependency Management](#7-renovate--dependency-management)
- [8. CTS Pipeline (Continuous Transport System)](#8-cts-pipeline-continuous-transport-system)
- [Pipeline Flow](#pipeline-flow)
- [Key References](#key-references)

---

## Environments

Each app has multiple Fly.io apps representing different environments. Environment-specific config lives in separate `fly.toml` files.

| Environment | Fly App Name | Config File | Trigger | Lifetime |
|-------------|-------------|-------------|---------|----------|
| **Preview** | `<app>-pr-<number>` | `fly.preview.toml` | PR opened/updated | Ephemeral -- destroyed on PR close |
| **Dev** | `<app>-dev` | `fly.dev.toml` | Push to `develop` / feature branch merge | Persistent |
| **Staging** | `<app>-staging` | `fly.staging.toml` | Push to `main` / release candidate | Persistent |
| **Production** | `<app>` | `fly.toml` | Release tag / manual promotion | Persistent |

### Environment Strategy

- **Preview (fleeting):** Spun up per PR using `superfly/fly-pr-review-apps`. Automatically destroyed when PR is closed/merged. Uses smaller machine specs. Lets reviewers click a link in the PR to see changes live.
- **Dev:** Shared development environment. Auto-deploys on merge to develop branch. May use seed/test data. Good for integration work across services.
- **Staging:** Mirror of production. Auto-deploys on merge to main. Used for final QA, stakeholder demos, and pre-release validation. Should match prod config (regions, scaling) as closely as possible.
- **Production:** Deployed on release tag creation or manual promotion from staging. May require approval gate.

### Secrets & Config (Doppler)

- All secrets are managed in **Doppler**, scoped by project and environment (`preview`, `dev`, `staging`, `production`).
- In CI, secrets are injected at runtime via `doppler run` with the appropriate environment config:
  ```bash
  doppler run --project myapp --config stg -- flyctl deploy -c fly.staging.toml
  ```
- No secrets are stored in GitHub, fly.toml, or environment variables directly. Doppler is the single source of truth.
- The `DOPPLER_TOKEN` (service token per environment) is the only secret stored in CI (as a GitHub Actions secret or CI variable).

---

## PR Management (Graphite)

All PRs follow the **stacked PR** workflow managed by [Graphite](https://graphite.dev). Stacking encourages small, focused PRs that are easier to review and merge incrementally. Graphite also manages CI optimization and merge queuing to control costs.

### Stacked PR Workflow

- Developers create stacks of dependent PRs using `gt create` / `gt stack submit`.
- Each PR in a stack targets the branch below it, not `main` directly.
- Graphite handles rebasing automatically when lower PRs in a stack are updated or merged.
- PRs merge from the bottom of the stack upward via Graphite's merge queue.

### CI Optimization

Stacking creates more PRs and more rebases, which can trigger redundant CI runs. Graphite's CI optimizer controls this by letting you configure **which PRs in a stack actually run CI**.

Every workflow starts with a Graphite CI optimizer job. Other jobs depend on its output:

```yaml
jobs:
  graphite_ci:
    runs-on: ubuntu-latest
    outputs:
      skip: ${{ steps.check.outputs.skip }}
    steps:
      - name: Graphite CI Optimizer
        id: check
        uses: withgraphite/graphite-ci-action@main
        with:
          graphite_token: ${{ secrets.GRAPHITE_CI_OPTIMIZER_TOKEN }}

  lint:
    needs: graphite_ci
    if: needs.graphite_ci.outputs.skip == 'false'
    # ...

  build:
    needs: [graphite_ci, lint]
    if: needs.graphite_ci.outputs.skip == 'false'
    # ...
```

**Configuration options (set in Graphite dashboard per repo):**

| Setting | Recommended | Effect |
|---------|-------------|--------|
| Run CI on bottom N PRs in stack | 1 | Only the PR about to merge runs full CI |
| Run CI on top of stack | Yes | Validates the full feature end-to-end |
| Skip mid-stack PRs | Yes | Avoids redundant runs on intermediate PRs |

**Cost optimization strategies:**

- **Fast vs. slow split:** Run lint on all PRs always; run E2E/deploy only on bottom + top of stack.
- **Merge queue batching:** Batch multiple stacks together so CI runs once per batch, not per PR.
- **Fail open:** If Graphite API errors, CI runs normally (never blocks merges).

### Merge Queue

- PRs are queued for merge via Graphite's merge queue (not GitHub's native one).
- Merge queue runs CI one final time before merging to `main`.
- Batch merges can combine multiple stacks, further reducing CI runs.

---

## 1. Lint

Static analysis and formatting checks. Runs first on every PR to catch issues cheaply before build/test. Lint runs on **all PRs** regardless of Graphite CI optimization (it's fast and cheap).

| Step | Tool(s) | Purpose |
|------|---------|---------|
| YAML Lint | `yamllint` | Validate YAML syntax and style across config files |
| Shell Lint | `shellcheck` | Lint shell scripts and CI workflow `run:` blocks |
| Commit Lint | `commitlint` + `@commitlint/config-conventional` | Enforce conventional commit format on PR titles for automated changelogs |
| License Check | `addlicense` (Google) | Verify SPDX license headers in source files |
| Dockerfile Lint | `hadolint` | Dockerfile best practices and security |
| Markdown Lint | `markdownlint` | Docs formatting consistency |
| Code Format | `prettier` / `black` / `gofmt` / `rustfmt` | Language-specific code formatting checks |
| Type Check | `tsc --noEmit` / `mypy` / `pyright` | Static type checking (language-dependent) |
| Terraform / IaC Lint | `tflint` / `tfsec` | Lint infrastructure-as-code |

---

## 2. Build

Compile, package, and prepare deployable artifacts.

| Step | Tool(s) | Purpose |
|------|---------|---------|
| Container Image Build | Docker / Buildah / Kaniko | Build container images from Dockerfile |
| Multi-arch Build | `docker buildx` / matrix strategy | Build for amd64 and arm64 |
| Image Tag | Git SHA / semver / branch name | Tag images for traceability |
| Push to Registry | GHCR / Docker Hub / Fly Registry | Push built images to container registry |
| Dependency Install | `npm ci` / `pip install` / `go mod download` | Install and cache dependencies |
| Application Build | `npm run build` / `go build` / `cargo build` | Compile application artifacts |
| SBOM Generation | Syft / Trivy | Generate Software Bill of Materials during build |

---

## 3. Scan

Security scanning and compliance checks across code, containers, and dependencies.

| Step | Tool(s) | Purpose |
|------|---------|---------|
| SAST (Static Analysis) | SonarQube / Semgrep / CodeQL | Static application security testing on source code |
| Secret Detection | `gitleaks` / `trufflehog` | Detect hardcoded secrets and credentials |
| CVE / Container Scan | Trivy / `grype` (Anchore) | Scan container images for known CVEs |
| Dependency Scan | `npm audit` / `pip-audit` / Snyk | Check dependencies for known vulnerabilities |
| IaC Security Scan | `checkov` / `tfsec` | Scan infrastructure-as-code for misconfigurations |
| License Compliance | OWASP Dependency-Track / `license-checker` | Enforce license policies against dependencies |
| SBOM Policy Check | OWASP Dependency-Track | Enforce vulnerability and license policies against SBOMs |
| Supply Chain Scorecard | OSSF Scorecard | Evaluate repo supply chain security posture |
| STIG Compliance | OpenSCAP / STIG benchmarks | Check against DISA STIG requirements (for gov deliverables) |
| Scan Report to PR | GitHub PR comment / SARIF upload | Post scan summary or diff as a PR comment for review |

---

## 4. Test

Validate functionality at multiple levels. Tests run against the fleeting preview environment where possible.

| Step | Tool(s) | Purpose |
|------|---------|---------|
| Unit Tests | Jest / pytest / Go test / etc. | Test individual functions and components |
| Integration Tests | Language/framework test runners | Test interactions between services, databases, APIs |
| E2E / Journey Tests | Playwright / Cypress | Test real user flows against a deployed preview app |
| API Tests | Postman / Newman / `httpyac` | Validate API contracts and responses |
| Smoke Test | curl / health check endpoints | Verify deployed app returns non-error responses |
| Performance / Load Test | k6 / Artillery | Baseline performance checks (optional, staging only) |
| Accessibility Test | `axe-core` / Lighthouse | Check for WCAG compliance (frontend apps) |
| Doc-Only Skip | File filter check | Skip heavy test jobs when only docs/markdown changed |

---

## 5. Deploy

Deploy to Fly.io across environments. Each environment is a separate Fly app with its own config.

| Step | Tool(s) | Purpose |
|------|---------|---------|
| **Preview Deploy** | `superfly/fly-pr-review-apps` | Ephemeral per-PR app on Fly.io, auto-created on PR open, destroyed on PR close |
| **Dev Deploy** | `doppler run --config dev -- flyctl deploy -c fly.dev.toml` | Auto-deploy to dev on push to develop branch |
| **Staging Deploy** | `doppler run --config stg -- flyctl deploy -c fly.staging.toml` | Auto-deploy to staging on push to main |
| **Production Deploy** | `doppler run --config prd -- flyctl deploy -c fly.toml` | Deploy to prod on release tag or manual approval |
| Inject Secrets | `doppler run --config <env>` | All secrets injected at runtime from Doppler -- no secrets in fly.toml or CI env vars |
| Run Migrations | `doppler run --config <env> -- flyctl ssh console -C "..."` | Run database migrations with secrets injected |
| Health Check | `flyctl status` / HTTP health endpoint | Verify the deploy is healthy before marking success |
| Rollback | `flyctl releases rollback` | Roll back to previous release on failure |
| Preview Cleanup | `superfly/fly-pr-review-apps` (on PR close) | Destroy ephemeral preview app and associated resources |

### Fly.io Deploy Config per Environment

```
project/
  fly.toml              # production
  fly.staging.toml      # staging
  fly.dev.toml          # dev
  fly.preview.toml      # PR previews (smaller machines, no HA)
```

Each file sets app name, region, machine size, scaling, and environment-specific services. Preview and dev use smaller VMs; staging mirrors production specs. Secrets are never in these files -- they come from Doppler at deploy time.

---

## 6. Release

Version management, changelogs, and promotion between environments.

| Step | Tool(s) | Purpose |
|------|---------|---------|
| Changelog Generation | `release-please` / `semantic-release` | Auto-generate changelogs from conventional commits |
| Version Bump | `release-please` / `semantic-release` | Bump version based on commit types (feat/fix/breaking) |
| Git Tag | `git tag` / release-please | Create semver git tags for releases |
| GitHub Release | `gh release create` / release-please | Create GitHub release with notes and artifacts |
| Container Tag | `docker tag` / registry API | Tag container image with release version |
| Production Promotion | Manual approval gate / GitHub Environment protection | Require approval before deploying release to production |
| Release Notification | Slack / Teams / webhook | Notify team of new release |

---

## 7. Renovate / Dependency Management

Automated dependency updates across the org.

| Step | Tool(s) | Purpose |
|------|---------|---------|
| Dependency Updates | Renovate Bot | Auto-update npm, pip, Go, Docker, Terraform, GitHub Actions, etc. |
| GitHub Action Pinning | `helpers:pinGitHubActionDigests` | Pin GitHub Actions to SHA digests for supply chain security |
| Dockerfile Base Image Updates | Renovate `docker` manager | Keep base images up to date |
| Auto-merge (patch) | Renovate `automerge` | Auto-merge passing patch-level updates to reduce toil |
| Scheduled Updates | Renovate `schedule` | Run updates on a schedule (e.g., weekly) to batch PRs |
| Group Updates | Renovate `packageRules` + `groupName` | Group related dependency updates into single PRs |
| Custom Managers | Renovate `customManagers` (regex) | Handle non-standard version patterns in config files |
| Dashboard | Renovate Dependency Dashboard issue | Central view of all pending updates and their status |

---

## 8. CTS Pipeline (Continuous Transport System)

For gov cloud / .mil deliverables: the pipeline for packaging, approving, and transporting artifacts to a secured or airgapped environment where ISSM approval is required.

> This component is only used when delivering to environments that require formal approval and/or airgap transport. It is not part of the standard dev/staging/prod flow.

### Artifact Preparation

| Step | Tool(s) | Purpose |
|------|---------|---------|
| Release Artifact Bundle | Docker export / tar / ORAS | Package all deployable artifacts into a transportable bundle |
| SBOM Export | Syft / Trivy (CycloneDX or SPDX format) | Generate SBOM for ISSM review |
| Vulnerability Report | Trivy / Grype | Generate final CVE scan report for the approval package |
| STIG Checklist | OpenSCAP / STIG Viewer | Produce STIG compliance artifacts for accreditation |
| Transport Manifest | Custom | Document contents, versions, and checksums (SHA256) for chain-of-custody |

### Approval Gate

| Step | Tool(s) | Purpose |
|------|---------|---------|
| ISSM Review | Manual / workflow gate | ISSM reviews SBOM + scan results + STIG checklist before authorizing transport |
| Approval Artifact | Signed approval record | Documented authorization to proceed with transport |

### Transport and Deployment

| Step | Tool(s) | Purpose |
|------|---------|---------|
| Media Transfer | Sneakernet / cross-domain diode | Physical media or cross-domain solution to move artifacts across the gap |
| High-Side Deploy | `flyctl deploy` / Docker import / platform-specific | Deploy to the secured target environment |
| Deployment Verification | Smoke tests / health checks | Verify the deployment is functional in the target environment |

---

## Pipeline Flow

```
Stacked PR Created / Updated (Graphite):
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  Graphite CI Optimizer ──> should this PR run CI?       │
  │       │                                                 │
  │       ├── mid-stack PR ──> skip (lint only)             │
  │       │                                                 │
  │       └── bottom/top of stack ──> run full pipeline:    │
  │           lint ──> build ──> scan ──> test               │
  │                                        │                │
  │                                        ▼                │
  │                             deploy preview (fleeting)    │
  │                             pr-123-myorg-myapp.fly.dev   │
  │                                        │                │
  │                                        ▼                │
  │                             E2E tests against preview    │
  └─────────────────────────────────────────────────────────┘
                                           │
Graphite Merge Queue (bottom of stack):    ▼
  ┌─────────────────────────────────────────────────────────┐
  │  final CI run ──> merge to main                         │
  │  (batched across stacks to reduce runs)                 │
  └─────────────────────────────────────────────────────────┘
                                           │
Merged to develop:                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │  deploy to dev (myapp-dev.fly.dev)                      │
  └─────────────────────────────────────────────────────────┘
                                           │
Merged to main:                            ▼
  ┌─────────────────────────────────────────────────────────┐
  │  deploy to staging (myapp-staging.fly.dev)              │
  │  run full E2E / smoke tests against staging             │
  └─────────────────────────────────────────────────────────┘
                                           │
Release Tag / Manual Promotion:            ▼
  ┌─────────────────────────────────────────────────────────┐
  │  release (changelog, git tag, GitHub release)           │
  │  deploy to production (myapp.fly.dev)                   │
  │  smoke test production                                  │
  │  notify team                                            │
  └─────────────────────────────────────────────────────────┘
                                           │
Scheduled:                                 ▼
  ┌─────────────────────────────────────────────────────────┐
  │  renovate (dependency updates)                          │
  └─────────────────────────────────────────────────────────┘

Gov / .mil Delivery (CTS - as needed):
  ┌─────────────────────────────────────────────────────────┐
  │  artifact bundle ──> SBOM ──> vuln report ──> STIG      │
  │       │                                                 │
  │       ▼                                                 │
  │  ISSM approval gate ──> transport ──> high-side deploy  │
  └─────────────────────────────────────────────────────────┘
```

### PR Cleanup

When a PR is closed or merged, the preview app is automatically destroyed by the `fly-pr-review-apps` action (triggers on `pull_request: closed`).

---

## Key References

- [Graphite](https://graphite.dev) - Stacked PRs, merge queue, CI optimization
- [Graphite CI Optimizations](https://graphite.com/docs/stacking-and-ci) - Configuring CI skip logic for stacked PRs
- [withgraphite/graphite-ci-action](https://github.com/withgraphite/graphite-ci-action) - GitHub Action for CI optimizer
- [Doppler](https://docs.doppler.com/) - Secrets management across environments
- [Fly.io: Continuous Deployment with GitHub Actions](https://fly.io/docs/launch/continuous-deployment-with-github-actions/) - CD setup guide
- [Fly.io: Git Branch Preview Environments](https://fly.io/docs/blueprints/review-apps-guide/) - Ephemeral PR review apps
- [Fly.io: App Configuration (fly.toml)](https://fly.io/docs/reference/configuration/) - Config file reference
- [superfly/fly-pr-review-apps](https://github.com/superfly/fly-pr-review-apps) - GitHub Action for PR preview deploys
- [superfly/flyctl-actions](https://github.com/superfly/flyctl-actions) - GitHub Action for flyctl
- [GitLab CI/CD Components Catalog](https://gitlab.com/explore/catalog) - Reusable GitLab CI components
- [OSSF Scorecard](https://securityscorecards.dev/) - Supply chain security scoring
- [Renovate Bot](https://docs.renovatebot.com/) - Automated dependency updates
- [release-please](https://github.com/googleapis/release-please) - Automated releases from conventional commits
