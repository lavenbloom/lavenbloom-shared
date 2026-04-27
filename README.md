# lavenbloom-shared

> **Runbook & Developer Walkthrough** — Centralized reusable GitHub Actions workflow library for the Lavenbloom platform.

---

## Table of Contents

1. [Overview](#overview)
2. [Repository Structure](#repository-structure)
3. [Workflow Reference](#workflow-reference)
   - [ci-sast.yml — SonarQube SAST](#ci-sastyml--sonarqube-sast)
   - [ci-sca.yml — Snyk Dependency Scan](#ci-scayml--snyk-dependency-scan)
   - [ci-docker-build.yml — Trivy Container Scan](#ci-docker-buildyml--trivy-container-scan)
   - [ci-docker-publish.yml — Docker Build & Push](#ci-docker-publishyml--docker-build--push)
   - [cd-template.yml — GitOps Helm Update](#cd-templateyml--gitops-helm-update)
   - [ci-notify.yml — Email Failure Alerts](#ci-notifyyml--email-failure-alerts)
4. [Pipeline Flow Per Service](#pipeline-flow-per-service)
5. [Required Secrets](#required-secrets)
6. [How to Call These Workflows](#how-to-call-these-workflows)
7. [Adding a New Service](#adding-a-new-service)
8. [Troubleshooting](#troubleshooting)

---

## Overview

`lavenbloom-shared` is the **single source of truth for all CI/CD logic** across the Lavenbloom microservices platform. It contains 6 reusable GitHub Actions workflows that are called by each service's individual pipeline.

### Why a shared workflow library?

Instead of duplicating 100+ lines of CI/CD YAML in every service repository, each service's workflow file is a **thin caller (~127 lines)** that delegates all logic here. This means:

- **One change** to the shared workflow propagates to all 5 services instantly
- **Consistent security gates** — every service runs the same SAST, SCA, and container scan steps
- **No drift** between service pipelines

### Consuming services

| Service | Workflow file | Runtime |
|---|---|---|
| `auth-service` | `ci-auth-service.yml` | Python |
| `habit-service` | `ci-habit-service.yml` | Python |
| `journal-service` | `ci-journal-service.yml` | Python |
| `notification-service` | `ci-notification-service.yml` | Python |
| `frontend` | `ci-frontend.yml` | Node |

---

## Repository Structure

```
lavenbloom-shared/
└── .github/
    └── workflows/
        ├── ci-sast.yml            # SonarQube static analysis + quality gate
        ├── ci-sca.yml             # Snyk dependency scanning (Python & Node)
        ├── ci-docker-build.yml    # Temporary Docker build + Trivy CVE scan
        ├── ci-docker-publish.yml  # Docker tag, build, push + dedup check
        ├── cd-template.yml        # GitOps: update Helm values.yaml + push
        └── ci-notify.yml          # Email failure alerts via SendGrid
```

All workflows use `on: workflow_call` — they cannot be triggered directly. They must be called from another workflow using `uses:`.

---

## Workflow Reference

### `ci-sast.yml` — SonarQube SAST

**Purpose:** Static application security testing using SonarQube. Checks code quality, security hotspots, code smells, and coverage. Outputs a quality gate result for downstream decision-making.

**Special behavior for `auth-service`:** If `service-name == 'auth-service'`, runs `pytest --cov=app --cov-report=xml` before the SonarQube scan to upload coverage data.

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `service-name` | No | `""` | Service identifier — triggers pytest for `auth-service` |
| `service-path` | No | `.` | Path to service source (relative to repo root) |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `SONAR_TOKEN` | ✅ | SonarQube authentication token |
| `SONAR_URL` | ✅ | SonarQube server URL |

**Outputs:**

| Output | Description |
|---|---|
| `quality-gate` | `"success"` or `"failure"` — SonarQube quality gate result |

**Usage:**
```yaml
sast:
  uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-sast.yml@main
  with:
    service-name: auth-service
    service-path: .
  secrets:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_URL: ${{ secrets.SONAR_URL }}
```

---

### `ci-sca.yml` — Snyk Dependency Scan

**Purpose:** Software Composition Analysis using Snyk. Scans `requirements.txt` (Python) or `package.json` (Node) for known CVEs. Generates an HTML vulnerability report uploaded as an artifact.

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `service-name` | ✅ | — | Service name (used for artifact naming) |
| `service-path` | No | `.` | Path to service source |
| `runtime` | ✅ | — | `python` or `node` |
| `python-version` | No | `"3.11"` | Python version for Python scans |
| `node-version` | No | `"20"` | Node version for Node scans |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `SNYK_TOKEN` | ✅ | Snyk API authentication token |

**Artifacts produced:** `snyk-report-{service-name}` (HTML report, 14-day retention)

**Usage:**
```yaml
sca:
  needs: [sast]
  uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-sca.yml@main
  with:
    service-name: habit-service
    service-path: .
    runtime: python
  secrets:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

---

### `ci-docker-build.yml` — Trivy Container Scan

**Purpose:** Builds a **temporary Docker image** (never pushed to a registry) and runs a **Trivy vulnerability scan** against it. Fails the pipeline if CRITICAL or HIGH CVEs are found. Uploads the Trivy report as an artifact.

This workflow is the PR-time container security gate — it ensures no vulnerable image is ever published.

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `service-name` | ✅ | — | Service name (artifact naming) |
| `service-path` | No | `.` | Dockerfile location |

**Outputs:**

| Output | Description |
|---|---|
| `trivy-critical` | `"true"` if CRITICAL/HIGH CVEs found |

**Artifacts produced:** `trivy-report-{service-name}` (14-day retention)

**Usage:**
```yaml
trivy:
  needs: [sca]
  uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-docker-build.yml@main
  with:
    service-name: journal-service
    service-path: .
```

---

### `ci-docker-publish.yml` — Docker Build & Push

**Purpose:** The production image build and push workflow. It:
1. Generates the image tag (`dev-{SHA}` for develop, semver for releases)
2. Checks if the tag **already exists** in Docker Hub — skips the build if so (dedup protection)
3. Validates that a Git tag, if being created, doesn't point to a different commit (conflict prevention)
4. Builds the Docker image and pushes to Docker Hub
5. Creates and pushes the Git tag

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `service-name` | ✅ | — | Used to construct image name: `lavenbloom-{service-name}` |
| `service-path` | No | `.` | Dockerfile location |
| `environment` | ✅ | — | `develop` or `main` |
| `target-env` | ✅ | — | `dev` or `prod` (controls values file target) |
| `release-tag` | No | `""` | Semver tag from GitHub Release event |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `DOCKER_USERNAME` | ✅ | Docker Hub username |
| `DOCKER_PASSWORD` | ✅ | Docker Hub password or access token |

**Outputs:**

| Output | Description |
|---|---|
| `image-tag` | The tag that was built and pushed (passed to `cd-template.yml`) |

**Image naming convention:**

| Environment | Image name | Tag |
|---|---|---|
| Dev | `{DOCKER_USERNAME}/lavenbloom-{service-name}` | `dev-{7-char-SHA}` |
| Prod | `{DOCKER_USERNAME}/lavenbloom-{service-name}` | `v1.2.0` (from Release) |

**Usage:**
```yaml
dev-publish:
  uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-docker-publish.yml@main
  with:
    service-name: auth-service
    service-path: .
    environment: develop
    target-env: dev
  secrets:
    DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
    DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

---

### `cd-template.yml` — GitOps Helm Update

**Purpose:** Implements the GitOps continuous delivery pattern. After a Docker image is built and pushed, this workflow:
1. Clones the `lavenbloom-charts` Helm repository
2. Uses `yq` to update the `image.tag` field in the target `values-dev.yaml` or `values-prod.yaml`
3. Validates the YAML is still well-formed (`yq eval '.'`)
4. Commits and pushes the change to the charts repo
5. ArgoCD detects the commit via polling and automatically syncs the cluster

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `service-name` | ✅ | — | Helm chart folder name (e.g., `auth-service`) |
| `service-key` | ✅ | — | YAML root key in values file (e.g., `authService`) |
| `image-tag` | ✅ | — | New image tag to write into `image.tag` |
| `environment` | ✅ | — | Charts repo branch to push to (`develop` or `main`) |
| `target-env` | ✅ | `dev` | `dev` or `prod` — selects `values-dev.yaml` or `values-prod.yaml` |
| `charts-repo` | No | `lavenbloom/lavenbloom-charts` | GitHub org/repo for the charts repository |
| `charts-path` | No | `microservices` | Path prefix within the charts repo |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `HELM_REPO_PAT` | ✅ | GitHub PAT with push access to the charts repository |

**Commit message format:**
```
cd(auth-service): update image tag to dev-a3f9c12 in values-dev.yaml [develop]
```

**Usage:**
```yaml
dev-cd:
  needs: [dev-publish]
  uses: lavenbloom/lavenbloom-shared/.github/workflows/cd-template.yml@main
  with:
    service-name: auth-service
    service-key: authService
    image-tag: ${{ needs.dev-publish.outputs.image-tag }}
    environment: develop
    target-env: dev
  secrets:
    HELM_REPO_PAT: ${{ secrets.HELM_REPO_PAT }}
```

---

### `ci-notify.yml` — Email Failure Alerts

**Purpose:** Sends email failure notifications via SendGrid when one or more pipeline jobs have failed. The workflow only runs if at least one job result is `"failure"` — it silently skips on fully successful runs.

**Inputs:**

| Input | Required | Description |
|---|---|---|
| `service_name` | ✅ | Service name shown in the email subject |
| `git_tag` | No | Git tag or SHA for context |
| `sonarqube_result` | No | Job result string: `"success"`, `"failure"`, or `"skipped"` |
| `snyk_result` | No | Same format |
| `generate_tag_result` | No | Same format |
| `build_result` | No | Same format |
| `trivy_result` | No | Same format |
| `push_result` | No | Same format |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `SENDGRID_API_KEY` | ✅ | SendGrid API key for email delivery |

**Email content includes:**
- Service name, branch, tag, actor, commit message, SHA
- Table of failed jobs with status
- Full job result table
- Direct link to the GitHub Actions run

**Usage:**
```yaml
notify:
  needs: [sast, sca, trivy, dev-publish, dev-cd]
  if: always()
  uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-notify.yml@main
  with:
    service_name: auth-service
    sonarqube_result: ${{ needs.sast.result }}
    snyk_result: ${{ needs.sca.result }}
    build_result: ${{ needs.dev-publish.result }}
  secrets:
    SENDGRID_API_KEY: ${{ secrets.SENDGRID_API_KEY }}
```

---

## Pipeline Flow Per Service

```
┌──────────────────────────────────────────────────────────────────┐
│  PULL REQUEST to develop / main                                   │
│                                                                    │
│  [sast] ──▶ [sca] ──▶ [trivy] ──▶ [pr-check]                    │
│  SonarQube   Snyk     Docker        Aggregated                    │
│              deps     build +       pass/fail                     │
│                       CVE scan      gate                          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  PUSH to develop                                                   │
│                                                                    │
│  [dev-publish] ──────────────▶ [dev-cd]                          │
│  Build image                   Update values-dev.yaml             │
│  Tag: dev-{SHA}                ArgoCD syncs dev cluster           │
│  Push to Docker Hub                                               │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  GITHUB RELEASE created                                            │
│                                                                    │
│  [publish] ──────────────────▶ [cd]                              │
│  Build image                   Update values-prod.yaml            │
│  Tag: v1.x.x (semver)          ArgoCD syncs prod cluster          │
│  Push to Docker Hub                                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## Required Secrets

These secrets must be configured in **every service repository** that calls these shared workflows. Navigate to: `Repository → Settings → Secrets and variables → Actions → New repository secret`.

| Secret | Used by | How to obtain |
|---|---|---|
| `SONAR_TOKEN` | `ci-sast.yml` | SonarQube → My Account → Security → Generate Token |
| `SONAR_URL` | `ci-sast.yml` | URL of your SonarQube server (e.g. `https://sonar.example.com`) |
| `SNYK_TOKEN` | `ci-sca.yml` | [Snyk dashboard](https://app.snyk.io) → Account Settings → API Token |
| `DOCKER_USERNAME` | `ci-docker-publish.yml` | Your Docker Hub username |
| `DOCKER_PASSWORD` | `ci-docker-publish.yml` | Docker Hub → Account Settings → Security → Access Tokens |
| `HELM_REPO_PAT` | `cd-template.yml` | GitHub → Settings → Developer settings → Personal access tokens → `repo` scope on `lavenbloom-charts` |
| `SENDGRID_API_KEY` | `ci-notify.yml` | [SendGrid dashboard](https://app.sendgrid.com) → Settings → API Keys |

---

## How to Call These Workflows

A minimal service workflow calling the full PR + develop pipeline:

```yaml
name: CI/CD — my-service

on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [develop, main]
  release:
    types: [created]

jobs:

  sast:
    if: github.event_name == 'pull_request'
    uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-sast.yml@main
    with:
      service-name: my-service
      service-path: .
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_URL: ${{ secrets.SONAR_URL }}

  sca:
    needs: [sast]
    if: github.event_name == 'pull_request'
    uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-sca.yml@main
    with:
      service-name: my-service
      service-path: .
      runtime: python   # or: node
    secrets:
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  trivy:
    needs: [sca]
    if: github.event_name == 'pull_request'
    uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-docker-build.yml@main
    with:
      service-name: my-service
      service-path: .

  pr-check:
    needs: [sast, sca, trivy]
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - run: echo "All checks passed"

  dev-publish:
    if: github.event_name == 'push' && github.ref == 'refs/heads/develop'
    uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-docker-publish.yml@main
    with:
      service-name: my-service
      service-path: .
      environment: develop
      target-env: dev
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

  dev-cd:
    needs: [dev-publish]
    if: needs.dev-publish.result == 'success'
    uses: lavenbloom/lavenbloom-shared/.github/workflows/cd-template.yml@main
    with:
      service-name: my-service
      service-key: myService
      image-tag: ${{ needs.dev-publish.outputs.image-tag }}
      environment: develop
      target-env: dev
    secrets:
      HELM_REPO_PAT: ${{ secrets.HELM_REPO_PAT }}
```

---

## Adding a New Service

To onboard a new microservice into the Lavenbloom CI/CD platform:

### Step 1 — Create the service workflow

Copy the template from any existing service (e.g., `habit-service/ci-habit-service.yml`) and update:
- `service-name:` → new service name (must match chart folder in `lavenbloom-charts`)
- `service-key:` → camelCase key used as the root in `values-dev.yaml` / `values-prod.yaml`
- `runtime:` → `python` or `node`

### Step 2 — Add secrets to the new repo

Add all 6 secrets listed in [Required Secrets](#required-secrets) to the new repository's GitHub Secrets.

### Step 3 — Add the Helm chart

In `lavenbloom-charts/microservices/`, create:
```
my-service/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── secret.yaml
├── Chart.yaml
├── values-dev.yaml
└── values-prod.yaml
```

Ensure `values-dev.yaml` follows the standard image structure:
```yaml
my-service:
  image:
    repository: lavenbloom-my-service
    tag: "1.0.0"
```

### Step 4 — Add sonar-project.properties

In the service root:
```properties
sonar.projectKey=lavenbloom-my-service
sonar.projectName=lavenbloom-my-service
sonar.sources=app
sonar.exclusions=**/tests/**
```

### Step 5 — Create ArgoCD Application

Add an entry to the ArgoCD ApplicationSet in `lavenbloom-charts/argocd/` pointing to the new chart.

---

## Troubleshooting

### Workflow not found: `lavenbloom/lavenbloom-shared/.github/workflows/ci-sast.yml@main`

- Ensure the `lavenbloom-shared` repo is **public** (or the calling repo has access via GitHub Enterprise)
- Verify the file name matches exactly (case-sensitive)
- Confirm the `@main` branch exists and has the workflow file

### CD workflow pushes but ArgoCD does not sync

- Verify the ArgoCD Application's `repoURL` points to `lavenbloom-charts`
- Check ArgoCD sync interval (default: 3 minutes). Force a sync: `argocd app sync <app-name>`
- Ensure the `environment` input matches the branch that ArgoCD monitors

### Trivy scan fails in CI but passes locally

The Trivy scan in CI uses the latest vulnerability database. A CVE may have been newly published. Check the Trivy report artifact in the GitHub Actions run for the specific CVE ID and update the affected dependency.

### `yq: command not found` in cd-template

The `cd-template.yml` installs `yq` via `mikefarah/yq@v4.44.1` action. Ensure GitHub Actions runner has internet access during the workflow run. If using a self-hosted runner, pre-install `yq`.

### SonarQube quality gate timeout

The `SonarSource/sonarqube-quality-gate-action` has a 5-minute timeout. If the SonarQube server is slow, increase the `timeout-minutes` in `ci-sast.yml` or check SonarQube server load.
