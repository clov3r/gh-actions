# gh-actions

Shared GitHub Actions pipeline for QQ microservices. One reusable workflow configured per-repo via `.github/pipeline.yml`.

## How It Works

Each repo has:
1. **`.github/pipeline.yml`** — declares what the service is (images, tests, deploy targets)
2. **Thin workflow files** — each ~10 lines, just triggers + environment name

The shared pipeline handles everything: testing, building, pushing, deploying, tagging, changelog, and releasing.

## Complete Example: Orchestrator (Most Complex Setup)

Orchestrator is the most complex service: 2 Docker images, Go + Node tests, bake-based E2E, 3-target production deploy, and ECR builds. Here's the full setup.

### `.github/pipeline.yml`

```yaml
service_name: orchestrator-bff
release_name: Orchestrator

images:
  - name: quickquack/orchestrator
    context: ./orchestrator
    dockerfile: Dockerfile
    build_config: ""              # Opt out of environment BUILD_CONFIG
  - name: quickquack/orchestrator-bff
    context: .
    dockerfile: bff/Dockerfile
    # No build_config — inherits from environment (staging.qa, staging.pp, etc.)

tests:
  go:
    version: "1.26"
    dirs: [orchestrator, bff]     # Runs as parallel matrix jobs
  node:
    dir: device-management-ui
    command: npm run build -- --configuration production
  e2e:
    setup: |
      docker buildx bake \
        --set "*.args.BUILD_VERSION=test-${PIPELINE_SHA}" \
        --set "*.args.BUILD_GITHASH=${PIPELINE_SHA}" \
        --load e2e
    command: |
      export COMPOSE_BAKE=true
      ./run ci

# Override default deploy targets for production (3 targets instead of default 2)
deploy_environments:
  production:
    - env: preproduction
    - env: prod
      file: orchestrator-bff_beta_deployment.yaml
      label: BETA
      automerge: true
    - env: prod
      file: orchestrator-bff_v1_deployment.yaml
      label: PROD
      description: "Please ensure that the release has been tested in the beta environment before merging this PR."
      automerge: false
```

### Workflow Files

All workflow files follow the same pattern — only the trigger and environment differ.

**PR** (`.github/workflows/pr.yaml`):
```yaml
name: Pull Request Tests
on:
  pull_request:
    branches: [main, release/next]
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  pipeline:
    if: github.event.pull_request.draft == false || contains(github.event.pull_request.labels.*.name, 'test-draft')
    uses: qqcw/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: pr
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GH_PACKAGE_TOKEN: ${{ secrets.GH_PACKAGE_TOKEN }}
      KEYCLOAK_CLIENT_SECRET: ${{ secrets.KEYCLOAK_CLIENT_SECRET }}
```

**Dev** (`.github/workflows/dev-ci.yaml`):
```yaml
name: Build (dev branch)
on:
  push:
    branches: [dev]

permissions:
  contents: write

jobs:
  pipeline:
    uses: qqcw/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: dev
    secrets: inherit
```

**QA / Staging** (`.github/workflows/staging-ci.yaml`):
```yaml
name: Build (QA branch)
on:
  push:
    branches: [release/next, deploy/staging]

permissions:
  contents: write

jobs:
  pipeline:
    uses: qqcw/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: qa
    secrets: inherit
```

**Release Candidate** (`.github/workflows/release-candidate-ci.yaml`):
```yaml
name: Build (Release Candidate)
on:
  push:
    branches: [rc-*]

permissions:
  contents: write

jobs:
  pipeline:
    uses: qqcw/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: preproduction
    secrets: inherit
```

**Production Release** (`.github/workflows/main-ci.yaml`):
```yaml
name: Tag, Build and Release
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      version_override:
        description: "Force a specific version (e.g., 1.0.0). Leave empty for auto-increment."
        required: false

permissions:
  contents: write
  pull-requests: write

jobs:
  pipeline:
    uses: qqcw/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: production
      version_override: ${{ github.event.inputs.version_override || '' }}
    secrets: inherit
```

**Test branches** (`.github/workflows/test-ci.yaml`):
```yaml
name: Test
on:
  push:
    branches: ["*test*"]

jobs:
  pipeline:
    uses: qqcw/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: pr
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GH_PACKAGE_TOKEN: ${{ secrets.GH_PACKAGE_TOKEN }}
      KEYCLOAK_CLIENT_SECRET: ${{ secrets.KEYCLOAK_CLIENT_SECRET }}
```

### What Each Environment Does

| Environment | Unit Tests | E2E | Node | Build | Deploy | Changelog | Release |
|-------------|-----------|-----|------|-------|--------|-----------|---------|
| `pr` | go-test(orchestrator, bff) | bake e2e | UI build check | Docker check (no push) | — | — | — |
| `dev` | — | — | — | Push to Docker Hub | flux_apps PR (dev) | — | — |
| `qa` | go-test(orchestrator, bff) | bake e2e | UI build check | Push to Docker Hub | flux_apps PR (qa) | In deploy PR | — |
| `preproduction` | go-test(orchestrator, bff) | bake e2e | UI build check | Push to Docker Hub | flux_apps PR (preprod) | In deploy PR | — |
| `production` | — | — | — | Push to Docker Hub | 3 flux_apps PRs (preprod, BETA, PROD) | PR + in deploy PRs | GitHub Release |

---

## Environment Behavior Matrix

| Environment | Unit Tests | E2E | Build | Deploy | Release |
|-------------|-----------|-----|-------|--------|---------|
| `pr` | Yes | Yes | Check only | No | No |
| `dev` | No | No | Push | Yes | No |
| `qa` | Yes | Yes | Push | Yes | No |
| `preproduction` | Yes | Yes | Push | Yes | No |
| `production` | No | No | Push | Yes | Yes |

**Rationale:** PRs are the quality gate — every merge to main has passed tests + e2e. Staging/RC re-run tests before updating shared environments. Dev is a fast path. Production trusts that PR checks already passed.

## Pipeline Config Reference

### `pipeline.yml` Schema

```yaml
# Required
service_name: qscm                # Used for deploy paths, PR titles, image matching
release_name: QSCM                # Human-friendly name for GitHub Releases, Jira versions

# Required — Docker images to build
images:
  - name: quickquack/qscm         # Docker Hub image name
    context: .                     # Build context (default: .)
    dockerfile: Dockerfile         # Dockerfile path relative to context (default: Dockerfile)
    build_config: ""               # Explicit empty string opts OUT of environment BUILD_CONFIG
                                   # Omit entirely to inherit environment default (staging.qa, etc.)

# Optional — test configuration (runs when environment enables tests)
tests:
  go:
    version: "1.25"                # Go version
    dirs: [server]                 # Directories with go.mod — each runs as parallel matrix job
    timeout: 120s                  # Test timeout (default: 120s)

  node:
    dir: client                    # Directory with package.json
    node_version: "24"             # Node version (default: 24)
    command: ""                    # Custom command (default: npm test -- --watch=false --browsers=ChromeHeadless)

  e2e:
    setup: ""                      # Pre-test setup (e.g., docker buildx bake --load e2e)
    command: ./run ci              # E2E test command
    cleanup: ""                    # Cleanup command (runs on always())
    logs_command: ""               # Show logs on failure

  extra:                           # Additional checks (swagger, mocks, lint) — runs as parallel matrix
    - name: Check Swagger          # Human-readable name (appears in GitHub check name)
      setup: go install ...        # Optional setup command
      command: cd server && ...    # Check command — exit 0 = pass, non-zero = fail
      error: "Swagger docs stale"  # Error message shown on failure

# Optional — override default deploy targets per environment
# Omit to use defaults. Set to [] to skip deployment entirely.
deploy_environments:
  dev: []                          # No flux_apps deploy (e.g., deployed via ansible)
  production:
    - env: preproduction
    - env: prod
      file: my-svc_beta_deployment.yaml   # Custom deployment file in flux_apps
      label: BETA                          # Human-readable label for PR title
      description: "Test in beta first."   # Extra text in PR body
      automerge: true                      # Add automerge label (default: true)
    - env: prod
      file: my-svc_v1_deployment.yaml
      label: PROD
      automerge: false
```

### E2E Environment Variables

E2E commands can't use `${{ }}` expressions (config files are plain text). The pipeline provides these env vars:

| Variable | Value |
|----------|-------|
| `PIPELINE_SHA` | `github.sha` |
| `PIPELINE_REF` | `github.ref_name` |
| `PIPELINE_HEAD_REF` | `github.head_ref` (PR source branch) or `github.ref_name` |
| `PIPELINE_PR_NUMBER` | `github.event.number` (empty on push) |

### Default Deploy Targets

If `deploy_environments` is not specified for an environment:

| Environment | Default Targets |
|-------------|----------------|
| `pr` | None |
| `dev` | `[{env: dev}]` |
| `qa` | `[{env: qa}]` |
| `preproduction` | `[{env: preproduction}]` |
| `production` | `[{env: preproduction}, {env: prod}]` |

Deploy files default to `apps/{env}/{service_name}/{service_name}_deployment.yaml` in `qqcw/flux_apps`.

### Default Build Config per Environment

| Environment | `BUILD_CONFIG` injected | Extra Docker Tag |
|-------------|------------------------|-----------------|
| `pr` | — | — |
| `dev` | — | `latest-dev` |
| `qa` | `staging.qa` | `latest-qa` |
| `preproduction` | `staging.pp` | `latest-rc` |
| `production` | — | `latest` |

Images with `build_config: ""` explicitly set will NOT receive the environment default.

### Docker Build Args

The pipeline always passes these build args (unused args are silently ignored by Docker):

| Build Arg | Value | Notes |
|-----------|-------|-------|
| `BUILD_VERSION` | Version tag | Standard naming |
| `VERSION` | Version tag | Alias for legacy Dockerfiles |
| `BUILD_GITHASH` | Git SHA | Standard naming |
| `GIT_HASH` | Git SHA | Alias for legacy Dockerfiles |
| `BUILD_TIME` | Commit timestamp | |
| `BRANCH` | Branch name | |
| `BUILD_CONFIG` | Environment config | Only if set (per-image or environment default) |
| `GH_PACKAGE_TOKEN` | npm token | Only if secret is provided |

## Secrets

### PR / Test Workflows — Pass Explicit Secrets

```yaml
secrets:
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
  GH_PACKAGE_TOKEN: ${{ secrets.GH_PACKAGE_TOKEN }}
  # Add KEYCLOAK_CLIENT_SECRET only if the repo has e2e tests needing auth
```

### Deploy Workflows (dev, qa, preproduction, production) — Inherit All

```yaml
secrets: inherit
```

### Required Secrets

| Secret | Used For | Environments |
|--------|----------|-------------|
| `DOCKER_USERNAME` | Docker Hub login | All |
| `DOCKER_PASSWORD` | Docker Hub login | All |
| `PR_OPENER` | flux_apps deploy PRs + git tagging (PAT, triggers ECR workflows) | Deploy envs |
| `GH_PACKAGE_TOKEN` | @qqcw npm packages | Repos with Node dependencies |
| `KEYCLOAK_CLIENT_SECRET` | E2E auth | Repos with auth-dependent e2e |

### Why PR_OPENER for Tagging?

Tags created with `GITHUB_TOKEN` don't trigger other workflows (GitHub prevents infinite loops). Tags created with a PAT (`PR_OPENER`) DO trigger `on: push: tags:` workflows, which is required for tag-triggered ECR builds.

## ECR Builds

ECR workflows are NOT part of the shared pipeline. They remain in each repo and work via one of these patterns:

| Pattern | Repos | How It Works |
|---------|-------|-------------|
| **Tag-triggered** | QSCM, Orchestrator, QSoap Config, QSense Config | Pipeline creates tag with PAT → ECR workflow fires on `on: push: tags:` |
| **workflow_call** | Conducktor | Workflow chains ECR after pipeline: `needs: pipeline` with `tag: ${{ needs.pipeline.outputs.version }}` |
| **Branch-triggered** | QAlerts | Separate ECR workflow runs on same branch push alongside pipeline |

### Chaining ECR via workflow_call

For repos using `workflow_call` ECR (like conducktor), the deploy workflow chains it:

```yaml
jobs:
  pipeline:
    uses: qqcw/gh-actions/.github/workflows/pipeline.yml@v1
    with:
      environment: dev
    secrets: inherit

  ecr:
    needs: pipeline
    uses: ./.github/workflows/build-ecr-dev.yaml
    with:
      tag: ${{ needs.pipeline.outputs.version }}
    permissions:
      id-token: write
      contents: read
```

## Permissions

### PR Workflows

Default permissions (contents: read) are sufficient. Do NOT add `permissions:` block.

### Deploy Workflows (dev, qa, preproduction, production)

```yaml
permissions:
  contents: write          # For git tagging
```

### Production Release Workflows

```yaml
permissions:
  contents: write          # For git tagging + changelog PR
  pull-requests: write     # For changelog PR
```

### Why Not Job-Level Permissions in the Pipeline?

GitHub validates job-level permissions in reusable workflows against the caller's event grants BEFORE evaluating `if` conditions. A `pull_request` event only allows `contents: read`, so any job requesting `contents: write` — even if skipped — causes a `startup_failure`. Permissions must be set at the caller level.

## Branch Protection

### Recommended Required Status Checks

All checks are prefixed with `pipeline / `:

| Repo Type | Required Checks |
|-----------|----------------|
| Go only | `go-test (.)`, `build-check` |
| Go (multi-dir) | `go-test (server)`, `build-check` |
| Go + Node | `go-test (server)`, `node-test`, `build-check` |
| Go + E2E | `go-test (server)`, `e2e`, `build-check` |
| Go + E2E + Extra | `go-test (server)`, `e2e`, `extra-checks (0, ...)`, `extra-checks (1, ...)`, `build-check` |

Skipped jobs (e.g., `deploy`, `release`, `version`) should NOT be required — they only run on deploy branches.

## Migration Checklist

1. Create `.github/pipeline.yml` with your service config
2. Replace each workflow file with the thin trigger version
3. For PR/test workflows: pass explicit secrets (not `secrets: inherit`)
4. For deploy workflows: add `permissions: contents: write` and use `secrets: inherit`
5. For production: add `permissions: contents: write` + `pull-requests: write`
6. Verify branch triggers match the originals exactly
7. If no flux_apps deploy: set `deploy_environments` to `[]` for all envs
8. Test on a PR branch first (`environment: pr`)
9. Test each deploy environment (dev, qa, preproduction)
10. Set up required status checks in branch protection
11. Verify ECR builds still fire (tag-triggered repos need PAT for tagging)

## Branch Sync Workflow

Keeps downstream branches (e.g. `release/next`, `dev`) in sync with `main` after each push. It replays all commits unique to the branch on top of the latest `main` — merge commits are cherry-picked with `-m 1` to preserve PR diffs, regular commits are cherry-picked directly. If any replay fails (conflict), the job aborts with no changes pushed.

### Caller Workflow (`.github/workflows/sync-branches.yaml`)

```yaml
name: Sync branches
on:
  push:
    branches: [main]

jobs:
  sync:
    uses: qqcw/gh-actions/.github/workflows/sync-branches.yml@v1
    with:
      branches: '["release/next", "dev"]'
    secrets:
      PR_OPENER: ${{ secrets.PR_OPENER }}
```

### Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `branches` | JSON string array | `["release/next", "dev"]` | Branches to replay onto main |

### Behavior

1. Finds all commits on the branch not reachable from `main` (`git rev-list --reverse main..branch`)
2. Creates a new branch from `main`
3. Replays each commit in order: cherry-pick for regular commits, cherry-pick `-m 1` for merge commits
4. Force-pushes the result (with `--force-with-lease`)

- Each branch is synced independently (parallel, `fail-fast: false`)
- If a branch doesn't exist, it's skipped with a warning
- If a branch already contains all of `main`, it's a no-op
- If any cherry-pick fails (conflict), it aborts immediately — **no changes are pushed**

### Branch Protection

The target branches (`release/next`, `dev`) must **not** have the `non_fast_forward` rule, since this workflow force-pushes. The `pull_request` rule alone is sufficient to prevent humans from pushing directly — force-push is only used by this CI automation via `PR_OPENER`.

## Gate Main Workflow

Auto-approves PRs from `release/*` branches into `main` (so they can merge without manual approval). Closes PRs from any other branch with a comment directing them to `release/next`.

### Caller Workflow (`.github/workflows/gate-main.yaml`)

```yaml
name: Gate main
on:
  pull_request:
    branches: [main]
    types: [opened]

jobs:
  gate:
    uses: qqcw/gh-actions/.github/workflows/gate-main.yml@v1
    secrets:
      PR_OPENER: ${{ secrets.PR_OPENER }}
```

### How It Works

- PR from `release/next` → `main`: auto-approved by `qqcw-ite` via `PR_OPENER`
- PR from any other branch → `main`: closed with a comment

This pairs with requiring >=1 approval on `main` — release PRs get the bot approval automatically, while direct PRs are rejected.

## Versioning

- Pin to major version: `@v1`
- Breaking changes bump major version
- Bug fixes and improvements auto-propagate to all consumers
- To update v1 tag: `git tag -f v1 && git push origin v1 -f && git push qqcw v1 -f`
