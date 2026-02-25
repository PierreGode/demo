# Tag-Triggered GitHub Actions Pipelines

This repository demonstrates how to use **Git tags** to trigger different GitHub Actions workflows. Each workflow listens for a specific tag pattern and runs a distinct pipeline, showcasing real-world scenarios.

## Why Tag-Triggered Pipelines?

Tags are immutable references to a specific commit. Unlike branch-based triggers, tag-based pipelines let you:

- **Decouple releases from branches** — push a tag when you're ready, not when code lands on `main`
- **Encode metadata in the tag name** — version numbers, environments, issue IDs
- **Gate deployments** — different tag prefixes route to different environments
- **Create audit trails** — every release/deploy/hotfix is a named, permanent marker in Git history

---

## Workflow Overview

| # | Workflow | File | Tag Pattern | Example Tags | What It Demonstrates |
|---|---------|------|-------------|-------------|---------------------|
| 1 | **Semantic Release** | `tag-release.yml` | `v*` | `v1.0.0`, `v2.1.0-beta` | Stable vs prerelease conditional jobs |
| 2 | **Environment Deploy** | `tag-deploy.yml` | `deploy-*` | `deploy-dev-1`, `deploy-prod-42` | Tag-prefix routing to GitHub Environments |
| 3 | **Hotfix Matrix** | `tag-hotfix.yml` | `hotfix-*` | `hotfix-issue-123` | Matrix testing across OS + Node versions |
| 4 | **Changelog & Release** | `tag-changelog.yml` | `release-*` | `release-2024.02` | Auto-generated release notes + GitHub Release |

---

## Workflow Details

### 1. Semantic Release (`tag-release.yml`)

**Trigger:** Any tag starting with `v` (e.g. `v1.0.0`, `v3.2.1-rc.1`)

**How it works:**
- The `build` job runs on every `v*` tag and parses the tag to determine if it is a stable version (`v1.0.0`) or a prerelease (`v1.0.0-beta`)
- If **stable** — the `release` job runs (publish artefacts, deploy to production)
- If **prerelease** — the `prerelease` job runs (deploy to staging, publish beta package)

**Key concepts:** Job outputs, conditional `if` expressions, `GITHUB_REF_NAME`

```bash
git tag v1.0.0 && git push origin v1.0.0           # triggers build + release
git tag v1.1.0-beta && git push origin v1.1.0-beta  # triggers build + prerelease
```

---

### 2. Environment Deploy (`tag-deploy.yml`)

**Trigger:** Any tag starting with `deploy-` (e.g. `deploy-staging-42`, `deploy-prod-100`)

**How it works:**
- The `resolve-env` job parses the tag to extract the target environment and deploy ID
- Tag `deploy-staging-42` resolves to environment=`staging`, deploy_id=`42`
- Three conditional deploy jobs exist: `deploy-dev`, `deploy-staging`, `deploy-prod`
- Only the job matching the parsed environment runs
- Each job targets a GitHub Environment (which can have approval rules and secrets)

**Key concepts:** Tag prefix routing, GitHub Environments, environment protection rules

```bash
git tag deploy-dev-1 && git push origin deploy-dev-1             # deploys to dev (no approval)
git tag deploy-staging-42 && git push origin deploy-staging-42   # deploys to staging
git tag deploy-prod-100 && git push origin deploy-prod-100       # deploys to production
```

> **Tip:** Configure environment protection rules in your repository settings under **Settings > Environments** to require manual approval for `staging` and `production`.

---

### 3. Hotfix Matrix (`tag-hotfix.yml`)

**Trigger:** Any tag starting with `hotfix-` (e.g. `hotfix-issue-123`, `hotfix-login-crash`)

**How it works:**
- The `test-matrix` job runs tests across a 3x3 matrix:
  - **OS:** Ubuntu, Windows, macOS
  - **Node.js:** 18, 20, 22
- All 9 combinations must pass (`fail-fast: false` ensures every cell runs)
- Only when all matrix cells succeed does the `patch` job run

**Key concepts:** Matrix strategy, cross-platform testing, `fail-fast`, tag-gated patching

```bash
git tag hotfix-issue-123 && git push origin hotfix-issue-123
```

---

### 4. Changelog & GitHub Release (`tag-changelog.yml`)

**Trigger:** Any tag starting with `release-` (e.g. `release-2024.02`, `release-sprint-42`)

**How it works:**
- Checks out the full Git history (`fetch-depth: 0`)
- Finds the **previous tag** by commit date
- Generates a changelog from all commits between the previous and current tag
- Creates a **GitHub Release** with the changelog as the body

**Key concepts:** `git log` range between tags, `gh release create`, `permissions: contents: write`, calendar versioning

```bash
git tag release-2024.02 && git push origin release-2024.02
```

> **Note:** This workflow requires the `GITHUB_TOKEN` to have write access to contents (configured via `permissions` in the workflow).

---

## Quick Reference: How Tag Triggers Work in GitHub Actions

```yaml
on:
  push:
    tags:
      - "v*"          # matches v1.0.0, v2.3.1-beta, v-anything
      - "deploy-*"    # matches deploy-prod-1, deploy-staging-42
```

- `on.push.tags` fires when a tag matching the pattern is pushed to the remote
- The tag name is available as `GITHUB_REF_NAME` (e.g. `v1.0.0`)
- The full ref is `GITHUB_REF` (e.g. `refs/tags/v1.0.0`)
- You can combine `tags` with `paths` — but note: path filters only apply to file changes, not the tag name itself
- Tags do **not** trigger `pull_request` events — only `push`

## Testing All Workflows

Run these commands to trigger each workflow:

```bash
# 1. Semantic Release — stable
git tag v1.0.0
git push origin v1.0.0

# 1. Semantic Release — prerelease
git tag v1.1.0-beta
git push origin v1.1.0-beta

# 2. Environment Deploy — dev / staging / prod
git tag deploy-dev-1
git push origin deploy-dev-1

git tag deploy-staging-42
git push origin deploy-staging-42

git tag deploy-prod-100
git push origin deploy-prod-100

# 3. Hotfix Matrix
git tag hotfix-issue-99
git push origin hotfix-issue-99

# 4. Changelog & GitHub Release
git tag release-2024.02
git push origin release-2024.02
```

Then check the **Actions** tab in your GitHub repository to see the workflows run.
