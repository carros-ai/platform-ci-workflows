# platform-ci-workflows

Reusable GitHub Actions workflows for all carros-ai services.

## Available workflows

| Workflow | Purpose |
|---|---|
| `go-ci.yml` | Go: lint (golangci-lint) + tests + coverage |
| `go-release.yml` | Go: CI + build Docker image + push ghcr.io + deploy Coolify |
| `python-ci.yml` | Python: ruff + pytest + coverage |
| `python-release.yml` | Python: CI + build Docker image + push ghcr.io + deploy Coolify |
| `node-ci.yml` | Node: eslint/biome + tests |
| `node-release.yml` | Node: CI + build Docker image + push ghcr.io + deploy Coolify |

## Two deploy scenarios

### Scenario 1 — Skill deploy (any branch)

Triggered manually via `workflow_dispatch` — the Claude Code skill commits, pushes, and runs this.

Image tags: `sha-{abc1234}` + `latest`

### Scenario 2 — Release (merge to main)

Triggered automatically on every merge to `main`. Auto-bumps patch version, creates git tag and GitHub Release.

Image tags: `sha-{abc1234}` + `1.2.3` + `1.2` + `1` + `latest`

---

## Adding CI/CD to a new service

### 1. Create `.github/workflows/ci.yml`

Runs on PRs and feature branch pushes (lint + tests only, no deploy).

**Go:**
```yaml
name: CI
on:
  pull_request:
  push:
    branches-ignore: [main]
jobs:
  ci:
    uses: carros-ai/platform-ci-workflows/.github/workflows/go-ci.yml@main
    with:
      coverage-threshold: 80 # override per-service while coverage is being built up
```

**Python:**
```yaml
name: CI
on:
  pull_request:
  push:
    branches-ignore: [main]
jobs:
  ci:
    uses: carros-ai/platform-ci-workflows/.github/workflows/python-ci.yml@main
```

**Node:**
```yaml
name: CI
on:
  pull_request:
  push:
    branches-ignore: [main]
jobs:
  ci:
    uses: carros-ai/platform-ci-workflows/.github/workflows/node-ci.yml@main
```

---

### 2. Create `.github/workflows/release.yml`

Handles both deploy scenarios with a single file.

**Go:**
```yaml
name: Release
on:
  workflow_dispatch:        # Scenario 1: skill deploy (any branch)
  push:
    branches: [main]        # Scenario 2: release on merge
jobs:
  release:
    uses: carros-ai/platform-ci-workflows/.github/workflows/go-release.yml@main
    with:
      app-name: my-service-name
      release: ${{ github.event_name == 'push' }}
      coverage-threshold: 80 # override per-service while coverage is being built up
    secrets: inherit
```

**Python:**
```yaml
name: Release
on:
  workflow_dispatch:
  push:
    branches: [main]
jobs:
  release:
    uses: carros-ai/platform-ci-workflows/.github/workflows/python-release.yml@main
    with:
      app-name: my-service-name
      release: ${{ github.event_name == 'push' }}
    secrets: inherit
```

**Node:**
```yaml
name: Release
on:
  workflow_dispatch:
  push:
    branches: [main]
jobs:
  release:
    uses: carros-ai/platform-ci-workflows/.github/workflows/node-release.yml@main
    with:
      app-name: my-service-name
      release: ${{ github.event_name == 'push' }}
    secrets: inherit
```

---

### 3. Add repository secrets

In the GitHub repo settings → Secrets → Actions:

| Secret | Value |
|---|---|
| `COOLIFY_WEBHOOK_URL` | Deploy webhook URL from the Coolify service panel |
| `COOLIFY_DEPLOY_TOKEN` | Coolify API/deploy token used as the webhook bearer token |
| `GH_TOKEN` | Optional PAT with access to private `carros-ai/*` modules/packages when `GITHUB_TOKEN` is not enough |

> The `GITHUB_TOKEN` for ghcr.io authentication is provided automatically. If the Coolify secrets are missing, release workflows still build, scan, sign, and publish the image, but skip deploy with a warning instead of failing at workflow startup.

---

### 4. Configure Coolify to pull from ghcr.io

In the Coolify service settings, set the image to:
```
ghcr.io/carros-ai/<app-name>:latest
```

The webhook triggers a pull + restart on every deploy.

---

## Image naming convention

```
ghcr.io/carros-ai/<app-name>:<tag>
```

| Tag | When |
|---|---|
| `latest` | Every deploy |
| `sha-abc1234` | Every deploy (short SHA for traceability) |
| `1.2.3` | Releases only (merge to main) |
| `1.2` | Releases only |
| `1` | Releases only |
