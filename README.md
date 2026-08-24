# gh-actions

Shared GitHub Actions for QQ microservices. One pipeline workflow, configured per-repo via `.github/pipeline.yml`.

## Quick Start

### 1. Add `.github/pipeline.yml` to your repo

```yaml
service_name: qscm
release_name: QSCM

images:
  - name: quickquack/qscm

tests:
  go:
    version: "1.25"
    dirs: [server]
```

### 2. Replace your workflow files

Each workflow file becomes a thin trigger → pipeline call:

```yaml
# .github/workflows/main-ci.yaml
name: Release
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      version_override:
        description: "Force version (e.g., 1.0.0)"
        required: false

jobs:
  pipeline:
    uses: clov3r/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: production
      version_override: ${{ inputs.version_override || '' }}
    secrets: inherit
```

```yaml
# .github/workflows/pr.yaml
name: PR Tests
on:
  pull_request:
    branches: [main, release/next]

jobs:
  pipeline:
    uses: clov3r/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: pr
    secrets: inherit
```

```yaml
# .github/workflows/staging-ci.yaml
name: QA
on:
  push:
    branches: [release/next, deploy/staging]

jobs:
  pipeline:
    uses: clov3r/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: qa
    secrets: inherit
```

```yaml
# .github/workflows/dev-ci.yaml
name: Dev
on:
  push:
    branches: [dev]

jobs:
  pipeline:
    uses: clov3r/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: dev
    secrets: inherit
```

```yaml
# .github/workflows/release-candidate-ci.yaml
name: RC
on:
  push:
    branches: [rc-*, deploy/preproduction]

jobs:
  pipeline:
    uses: clov3r/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: preproduction
    secrets: inherit
```

## Environment Behavior Matrix

The `environment` input controls what the pipeline does:

| Environment | Unit Tests | E2E | Build | Deploy | Release |
|-------------|-----------|-----|-------|--------|---------|
| `pr` | Yes | Yes | Check only | No | No |
| `dev` | No | No | Push | Yes (dev) | No |
| `qa` | Yes | Yes | Push | Yes (qa) | No |
| `preproduction` | Yes | Yes | Push | Yes (preprod) | No |
| `production` | No | No | Push | Yes (preprod+prod) | Yes |

**Rationale:** PRs are the quality gate (every merge has passed e2e). Staging/RC re-run tests before updating shared environments. Dev is a fast path. Production trusts that PR checks passed.

## Pipeline Config Reference

### `pipeline.yml` Schema

```yaml
# Required
service_name: qscm              # Used for deploy paths, PR titles
release_name: QSCM              # Human-friendly name for GitHub Releases

# Required — Docker images to build
images:
  - name: quickquack/qscm       # Docker Hub image name
    context: .                   # Build context (default: .)
    dockerfile: Dockerfile       # Dockerfile path relative to context (default: Dockerfile)
    build_config: ""             # Per-image BUILD_CONFIG override

# Optional — test configuration (runs when environment enables tests)
tests:
  go:
    version: "1.25"              # Go version
    dirs: [server]               # Directories with go.mod (matrix)
    timeout: 120s                # Test timeout (default: 120s)
  node:
    dir: client                  # Directory with package.json
    node_version: "24"           # Node version (default: 24)
    command: ""                  # Custom test command (default: npm test -- --watch=false --browsers=ChromeHeadless)
  e2e:
    setup: ""                    # Pre-test setup command (e.g., docker buildx bake)
    command: ./run ci            # E2E test command
    cleanup: ""                  # Cleanup command (runs on always())
    logs_command: ""             # Show logs on failure
  extra:                         # Additional checks (swagger, mocks, lint)
    - name: Check Swagger
      setup: go install github.com/swaggo/swag/cmd/swag@latest
      command: cd server && swag init && git diff --quiet server/docs
      error: "Swagger docs are stale"

# Optional — override default deploy targets per environment
deploy_environments:
  production:
    - env: preproduction
    - env: prod
      file: my-service_beta_deployment.yaml    # Custom deployment file name
      automerge: true                           # Add automerge label (default: true)
    - env: prod
      file: my-service_v1_deployment.yaml
      automerge: false
```

### Default Deploy Targets

If `deploy_environments` is not specified for an environment, these defaults apply:

| Environment | Default Deploy Targets |
|-------------|----------------------|
| `pr` | None |
| `dev` | `[{env: dev}]` |
| `qa` | `[{env: qa}]` |
| `preproduction` | `[{env: preproduction}]` |
| `production` | `[{env: preproduction}, {env: prod}]` |

Deploy files default to `apps/{env}/{service_name}/{service_name}_deployment.yaml` in `qqcw/flux_apps`.

### Default Build Config per Environment

| Environment | `BUILD_CONFIG` | Extra Docker Tag |
|-------------|---------------|-----------------|
| `pr` | — | — |
| `dev` | — | `latest-dev` |
| `qa` | `staging.qa` | `latest-qa` |
| `preproduction` | `staging.pp` | `latest-rc` |
| `production` | — | `latest` |

## Pipeline Inputs

The reusable workflow accepts only 2 inputs:

| Input | Required | Description |
|-------|----------|-------------|
| `environment` | Yes | `pr`, `dev`, `qa`, `preproduction`, `production` |
| `version_override` | No | Force a specific version (production + workflow_dispatch only) |

Everything else comes from `.github/pipeline.yml`.

## Required Secrets

Set these as repository secrets (or org-level):

| Secret | Used For | Required By |
|--------|----------|-------------|
| `DOCKER_USERNAME` | Docker Hub login | All environments |
| `DOCKER_PASSWORD` | Docker Hub login | All environments |
| `PR_OPENER` | Creating PRs in flux_apps | Deploy environments |
| `GH_PACKAGE_TOKEN` | @qqcw npm packages | Repos with Node dependencies |
| `KEYCLOAK_CLIENT_SECRET` | E2E auth | Repos with auth-dependent e2e |

Use `secrets: inherit` to pass all secrets through.

## Composite Actions

The pipeline uses these internally, but they can also be used standalone:

| Action | Description |
|--------|-------------|
| `actions/check-code-change` | Skip releases for changelog-only commits |
| `actions/version-resolve` | Semantic versioning with override support |
| `actions/changelog-generate` | Conventional commit changelog |
| `actions/docker-build-push` | Multi-image Docker builds |
| `actions/flux-deploy` | Deployment PRs in flux_apps |
| `actions/changelog-pr` | CHANGELOG.md update PRs |
| `actions/github-release` | GitHub Releases with deploy links |
| `actions/go-test` | Go unit tests with caching |

## Examples

See the `examples/` directory for complete migration examples:
- `examples/qscm/` — Simple single-image Go service with e2e and extra checks
- `examples/orchestrator/` — Multi-image service with bake-based e2e and 3-target prod deploy
- `examples/conducktor/` — Multi-image service, Go-only tests
- `examples/qsoap-config/` — Single-image with Go + Node tests

## Migration Checklist

1. Copy the appropriate `pipeline.yml` example to your repo's `.github/pipeline.yml`
2. Update `service_name`, `release_name`, `images`, and `tests` for your service
3. Replace each workflow file with the thin trigger version (see examples)
4. Ensure all required secrets are set
5. Test on a PR branch first (`environment: pr`)
6. Remove old workflow files once verified

## Versioning

- Pin to major version: `@v1`
- Breaking changes bump major version
- Bug fixes and improvements auto-propagate
