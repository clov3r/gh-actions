# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Shared GitHub Actions pipeline for all QQ microservices (`qqcw/gh-actions`). Consumer repos call this with a thin workflow file + a `.github/pipeline.yml` config declaring what the service needs. This repo has no build/test/lint commands of its own — it's pure GitHub Actions YAML.

## Architecture

### Three reusable workflows

- **`.github/workflows/pipeline.yml`** — The main CI/CD pipeline. Called by every QQ microservice with `environment: pr|dev|qa|preproduction|production`. One workflow handles all environments via a behavior matrix encoded in the `config` job's `flags` step.
- **`.github/workflows/ecr-build.yml`** — ECR image builds. Called by tag-triggered workflows in consumer repos. Reads `ecr.images[]` from pipeline.yml, merges with Docker Hub `images[]` via `from` field, builds via `qqcw/ecr-push-action@v2`. Environment derived from tag suffix or passed explicitly.
- **`.github/workflows/sync-branches.yml`** — Replays branch-unique commits on top of main via cherry-pick. Used to keep `release/next` and `dev` in sync after merges to main.

### Composite actions (`actions/`)

| Action | Purpose |
|--------|---------|
| `docker-build-push` | Builds/pushes Docker images from `images_json` config. Loops images, injects standard build args, uses buildx with GHA cache. |
| `version-resolve` | Dry-run `github-tag-action` to compute next semver, supports manual override and pre-release branches. |
| `changelog-generate` | Generates conventional-commit changelog between two git tags. Categorizes by `feat`, `fix`, `perf`, `ci`, `test`, `refactor`. |
| `changelog-pr` | Pushes CHANGELOG.md update directly to main (requires branch protection bypass actor). |
| `flux-deploy` | Creates PR in `qqcw/flux_apps` to update a deployment image reference. Uses `yaml-update-action`. |
| `github-release` | Creates GitHub Release with changelog + deployment PR links. |
| `go-test` | Runs Go tests with caching (standalone composite, though pipeline.yml also inlines Go test steps). |
| `check-code-change` | Skips release if only metadata files (CHANGELOG.md, .ai/) changed. |

### Pipeline flow (pipeline.yml)

```
config → check (production only) → version → build → deploy → release
                                  ↗ go-test ↗
                                  ↗ node-test ↗
                                  ↗ e2e ↗
                                  ↗ extra-checks ↗
```

- **config** job: parses `.github/pipeline.yml` from the caller repo, sets environment flags, enriches images with `build_config`
- **Tests** (go-test, node-test, e2e, extra-checks): run in parallel, gated by `run_tests` flag
- **version**: resolves semver via dry-run tag action; skipped for PR builds
- **build**: pushes Docker images; `build-check` (PR) runs `--load` instead of `--push`
- **deploy**: tags the version, creates flux_apps PRs for each deploy target
- **release**: generates changelog, creates changelog PR + GitHub Release (production only)

### Environment behavior matrix

| Env | Tests | Build | Deploy | Release |
|-----|-------|-------|--------|---------|
| `pr` | Yes | Check only (`--load`) | No | No |
| `dev` | No | Push | Yes | No |
| `qa` | Yes | Push + `BUILD_CONFIG=staging.qa` | Yes | No |
| `preproduction` | Yes | Push + `BUILD_CONFIG=staging.pp` | Yes | No |
| `production` | No | Push + `BUILD_CONFIG=production` | Yes (multi-target) | Yes |

### Consumer repo contract

Each consumer repo provides:
1. **`.github/pipeline.yml`** — declares `service_name`, `release_name`, `images`, `ecr.images`, `tests`, and optional `deploy_environments` overrides
2. **Thin workflow files** per trigger (pr, dev, qa, preproduction, production) — each ~10 lines, just sets `environment` and passes secrets
3. **ECR workflow files** (build-ecr-dev.yaml, build-ecr-prod.yaml) — thin callers to `ecr-build.yml`, triggered by tags

See `examples/` for reference configs (orchestrator is the most complex: multi-image, multi-dir Go tests, bake-based E2E, 3-target production deploy).

## Key Design Decisions

- **No job-level permissions in the reusable workflow**: GitHub validates permissions against caller event grants BEFORE `if` conditions. A `pull_request` event only allows `contents: read`, so any job with `contents: write` — even if skipped — causes `startup_failure`. Permissions must be set at caller level.
- **PAT (`PR_OPENER`) for tagging**: Tags created with `GITHUB_TOKEN` don't trigger other workflows. Tags from PAT DO trigger `on: push: tags:` workflows needed for ECR builds.
- **Deploy is inline in pipeline.yml** (not using the `flux-deploy` composite action): The deploy step loops through `deploy_envs_json` to create multiple flux_apps PRs in one job, which the composite action can't do.
- **`build_config` opt-out**: Images with `build_config: ""` explicitly set are NOT overridden by the environment default. Images that omit the key inherit the environment's BUILD_CONFIG. This applies to both Docker Hub and ECR images.
- **ECR `from` field**: ECR images in pipeline.yml link to Docker Hub images via `from: quickquack/my-service`, inheriting context, dockerfile, and build_config. The dockerfile path is joined from context + dockerfile and normalized to repo-relative (what `ecr-push-action` expects).
- **ECR deploy registries**: Deploy PRs use ECR URIs when `ecr.images[]` is configured. Dev/QA/PP environments use `377740805472.dkr.ecr.us-east-2.amazonaws.com`, prod uses `752897034123.dkr.ecr.us-east-2.amazonaws.com` (images mirrored from dev to prod automatically).
- **E2E env vars**: E2E config fields are plain text (not GitHub expressions), so the pipeline exposes `PIPELINE_SHA`, `PIPELINE_REF`, `PIPELINE_HEAD_REF`, `PIPELINE_PR_NUMBER` as env vars.

## Versioning and Releases

- Consumer repos pin to `@v1` (or `@main` during development)
- Force-update the v1 tag after changes: `git tag -f v1 && git push origin v1 -f && git push qqcw v1 -f`
- Breaking changes bump major version
- The `sync-branches.yml` workflow uses `--force-with-lease` to push; target branches must NOT have the `non_fast_forward` branch protection rule

## Conventions

- Commit messages use conventional commits (`feat:`, `fix:`, `ci:`, `docs:`, `refactor:`)
- The changelog generator parses these prefixes to categorize entries
- Deploy PRs in flux_apps use `ci: Release {service} {version} - {env}` format
- `automerge` label on deploy PRs triggers automatic merge in flux_apps
