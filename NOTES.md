# lavenbloom-shared — Study Notes

> Deep-dive study material for the CI/CD shared workflow library. Written for someone preparing to present this to a DevOps team.

---

## Table of Contents

1. [Why a Shared Workflow Library](#1-why-a-shared-workflow-library)
2. [GitHub Actions Core Concepts](#2-github-actions-core-concepts)
3. [ci-sast.yml — SonarQube SAST](#3-ci-sastyml--sonarqube-sast)
4. [ci-sca.yml — Snyk SCA](#4-ci-scayml--snyk-sca)
5. [ci-docker-build.yml — Trivy Container Scan](#5-ci-docker-buildyml--trivy-container-scan)
6. [ci-docker-publish.yml — Docker Build & Push](#6-ci-docker-publishyml--docker-build--push)
7. [cd-template.yml — GitOps Helm Update](#7-cd-templateyml--gitops-helm-update)
8. [ci-notify.yml — Failure Alerts](#8-ci-notifyyml--failure-alerts)
9. [Full Pipeline Flow](#9-full-pipeline-flow)
10. [Environment Management](#10-environment-management)
11. [How a DevOps Engineer Releases a New Version](#11-how-a-devops-engineer-releases-a-new-version)
12. [Q&A](#12-qa)

---

## 1. Why a Shared Workflow Library

### The real-world problem

Imagine 5 microservices, each with a CI pipeline. Without sharing, you copy-paste 150 lines of YAML into every repo. A security rule changes (e.g., Trivy must also fail on MEDIUM CVEs) — you update 5 files. Miss one? That service ships unprotected. This is called **pipeline drift**.

### What this library solves

All CI/CD logic lives **once** in this repo. Each service repo is a thin ~127-line *caller* that says "use that logic." One update here propagates to all services immediately.

### Business value

- **Consistency** — Every service runs the same security gates — no developer can skip a step
- **Speed** — Onboarding a new service takes minutes — copy the caller template, add secrets
- **Auditability** — Security teams audit one file, not five
- **Reduced cost** — Less YAML to maintain = less engineering time

### Alternatives and why this approach was chosen

| Approach | Problem |
|---|---|
| Copy-paste per repo | Pipeline drift, unmaintainable at scale |
| Jenkins Shared Library | Requires Jenkins infra, Groovy expertise |
| GitLab CI `include:` | GitLab-specific, this project is GitHub-hosted |
| GitHub Composite Actions | Can only share *steps*, not entire *jobs* — no parallel job orchestration |
| **GitHub Reusable Workflows** ✅ | Full job-level reuse, supports inputs/secrets/outputs, native GitHub feature |

**Reusable workflows vs composite actions:** Composite actions replace a set of steps within one job. Reusable workflows replace an entire job including the runner, environment, and job outputs. For a security gate that must be an isolated job (not part of the caller's job), reusable workflows are the right choice.

---

## 2. GitHub Actions Core Concepts

### `on: workflow_call`

Makes a workflow **callable** — it cannot trigger on its own. Another workflow must invoke it via `uses:`. Think of it as a function definition.

```yaml
on:
  workflow_call:
    inputs:    # Parameters the caller passes in (non-sensitive)
    secrets:   # Sensitive values (tokens, passwords — redacted in logs)
    outputs:   # Return values the caller reads after the job finishes
```

### How a caller invokes a reusable workflow

```yaml
jobs:
  sast:
    uses: lavenbloom/lavenbloom-shared/.github/workflows/ci-sast.yml@main
    with:
      service-name: auth-service   # passes an input
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  # passes a secret
```

The `@main` suffix pins the call to the `main` branch of the shared repo. This is important — if you pin to a SHA instead, you get immutable behavior but lose automatic updates.

### `$GITHUB_OUTPUT`

Modern way to pass data between steps:
```bash
echo "tag=dev-abc123" >> $GITHUB_OUTPUT
# Later step reads it as:
${{ steps.step_id.outputs.tag }}
```

### `needs:` and job ordering

```yaml
jobs:
  sca:
    needs: [sast]   # Waits for sast job to finish before starting
```
If `sast` fails, `sca` is skipped — unless the step has `if: always()`.

### `if: always()`

Forces a job/step to run even if prior jobs failed. Used in notification and cleanup jobs.

### `defaults: run: working-directory:`

Sets the shell working directory for all `run:` steps in the job. Avoids repeating `cd service-path` in every step.

---

## 3. ci-sast.yml — SonarQube SAST

**File:** `.github/workflows/ci-sast.yml`

### What problem does SAST solve?

**SAST = Static Application Security Testing.** Analyses source code *without running it* to find:
- Security vulnerabilities (SQL injection patterns, insecure crypto, hardcoded secrets)
- Code smells (overly complex functions, duplicated blocks)
- Bugs (unreachable code, null-pointer risks)
- Test coverage gaps

**Real-world scenario:** A developer writes `SECRET_KEY = "admin123"` directly in the source code. Without SAST, this ships to production. SonarQube flags it as a hardcoded credential at PR time, blocking the merge.

### Inputs

| Input | Required | Default | Purpose |
|---|---|---|---|
| `service-name` | No | `""` | When `auth-service`, triggers pytest+coverage before scan |
| `service-path` | No | `.` | Where the source code lives in the repo |

### Secrets

| Secret | Purpose |
|---|---|
| `SONAR_TOKEN` | Authenticates with SonarQube server API |
| `SONAR_URL` | URL of the SonarQube instance |

### Output

| Output | Value |
|---|---|
| `quality-gate` | `"success"` or `"failure"` — the SonarQube gate result |

### Every step explained

**`actions/checkout@v4` with `fetch-depth: 0`**

`fetch-depth: 0` fetches the **entire git history**, not just the latest commit. SonarQube uses this for:
- **Incremental analysis** — only reports issues introduced in *this* PR, not pre-existing issues
- **Blame data** — knows which developer introduced which line for reporting
- Without it, SonarQube treats every file as brand new and floods the PR with old issues

**Python + pytest setup (auth-service only)**

```yaml
if: inputs.service-name == 'auth-service'
```
Only auth-service currently has a test suite. This conditional block:
1. Sets up Python 3.10
2. Installs dependencies + pytest-cov + httpx
3. Runs `pytest --cov=app --cov-report=xml` → produces `coverage.xml`

SonarQube reads `coverage.xml` and shows exactly which lines in `app/` are covered or not. The quality gate can require a minimum coverage percentage (e.g., 80%).

**`SonarSource/sonarqube-scan-action@v6`**

Reads `sonar-project.properties` from the repo root and runs the scanner. The properties file tells SonarQube which files to analyse:
```properties
sonar.projectKey=lavenbloom-auth-service
sonar.sources=app
sonar.exclusions=**/tests/**
sonar.python.coverage.reportPaths=coverage.xml
```

**`SonarSource/sonarqube-quality-gate-action@v1`**

After uploading the analysis, **polls the SonarQube server** for up to 5 minutes waiting for the analysis to complete. Then checks: did the code pass all quality gate conditions? If not, this step fails, failing the entire job and blocking the PR.

**Set Gate Output (with `if: always()`)**

```bash
echo "status=${{ steps.gate.outcome }}" >> $GITHUB_OUTPUT
```
`if: always()` ensures the output is captured even when the gate failed — so the notification job can include this result in the failure email.

### What tools exist for SAST?

| Tool | Key feature | Cost |
|---|---|---|
| **SonarQube** ✅ | Unified dashboard, 30+ languages, quality gates | Free community / paid |
| **SonarCloud** | Cloud SaaS version of SonarQube | Free for public repos |
| **Semgrep** | Pattern-based, highly customizable rules | Free open source |
| **CodeQL** | GitHub-native, deep semantic analysis | Free for public repos |
| **Checkmarx / Veracode** | Enterprise-grade, expensive | Paid |
| **Bandit** | Python-only, lightweight | Free |

**Why SonarQube?** Single platform that handles both Python (backend) and TypeScript (frontend). Quality gate is configurable and shows historical trends over time — not just pass/fail, but *improvement/regression* from sprint to sprint. Integrates natively with the GitHub PR diff via the scan action.

---

## 4. ci-sca.yml — Snyk SCA

**File:** `.github/workflows/ci-sca.yml`

### What problem does SCA solve?

**SCA = Software Composition Analysis.** Modern apps are mostly third-party libraries. SCA tools maintain CVE databases and check your `requirements.txt` or `package.json` against known vulnerabilities.

**Real-world scenario:** A critical vulnerability (e.g., CVE-2023-XXXXX) is discovered in `cryptography==38.0.0`. Your `requirements.txt` pins this version. Snyk catches it at PR time, shows the fix (`upgrade to 41.0.0`), and blocks merge. Without SCA, this ships to production.

### Key logic decisions in this workflow

**Why `|| true` after snyk test?**

```bash
snyk test ... --json-file-output=snyk-report.json || true
```
`snyk test` exits with code `1` if vulnerabilities are found. Without `|| true`, the step would fail *immediately*, before the HTML report is uploaded. We need the report to be available for developers to review. So we suppress the exit code here and do our own check in the final step using `jq`.

**Why `npm ci` instead of `npm install`?**

`npm ci` (clean install) is stricter:
- Reads `package-lock.json` exactly — installs the same versions every time (reproducible)
- Fails if `package-lock.json` is out of sync with `package.json` (catches version drift)
- Never modifies `package-lock.json` (safe in CI)
- Faster in CI because it skips dependency resolution

**`jq` — the JSON processor**

```bash
CRITICAL_COUNT=$(jq '[.vulnerabilities[] | select(.severity == "critical" or .severity == "high")] | length' snyk-report.json)
```
`jq` is a command-line JSON processor pre-installed on GitHub-hosted runners. This command:
1. Opens `snyk-report.json`
2. Goes into the `vulnerabilities` array
3. `select()` filters only HIGH or CRITICAL items
4. `length` counts them
5. Stores the count in `CRITICAL_COUNT`

If count > 0: sets output `critical_found=true`. This is passed back to the caller workflow.

**Artifact retention: 14 days**

The HTML report is available in the GitHub Actions UI for 14 days. After that it's auto-deleted. DevOps teams typically review these during sprint retrospectives or security reviews.

### Alternatives to Snyk

| Tool | Strengths | Weakness |
|---|---|---|
| **Snyk** ✅ | Developer-friendly, actionable fix advice, multi-runtime | Free tier rate-limited |
| **Dependabot** | Native GitHub, auto-creates upgrade PRs | Only alerts, no rich reports |
| **OWASP Dependency-Check** | Free, comprehensive NVD database | Slow, lots of false positives |
| **pip-audit** | Lightweight, Python-only | No Node support |
| **JFrog Xray** | Enterprise, deep Artifactory integration | Paid, complex setup |

---

## 5. ci-docker-build.yml — Trivy Container Scan

**File:** `.github/workflows/ci-docker-build.yml`

### What problem does this solve?

Even if your Python code is clean and your pip packages are vulnerability-free, the **base Docker image** (`python:3.11-slim`) contains an OS (Debian) with packages like `openssl`, `libc`, `curl`. These OS packages can have CVEs that SCA misses entirely.

**Real-world scenario:** CVE-2022-0778 (OpenSSL infinite loop) — your base image includes a vulnerable OpenSSL. Trivy catches it at PR time. Without container scanning, this ships in every Docker image you publish.

### The "never push" contract

This workflow builds a Docker image tagged `<service-name>:pr-test` that exists **only on the GitHub Actions runner** during this job. It is never pushed to any registry. The runner is ephemeral — discarded after the job. This is a pure security gate.

### Step-by-step logic

**Build step:**
```yaml
docker build -t ${{ inputs.service-name }}:pr-test .
```
Runs the full Dockerfile — all layers. If the Dockerfile itself has a bug (syntax error, missing file, invalid command), it fails here, catching Dockerfile problems at PR time too.

**Trivy action:**
```yaml
uses: aquasecurity/trivy-action@master
with:
  image-ref: "${{ inputs.service-name }}:pr-test"
  format: "table"
  output: "${{ github.workspace }}/trivy-report-${{ inputs.service-name }}.txt"
  scan-type: "image"
  severity: "CRITICAL,HIGH"
  exit-code: "0"        # Don't auto-fail — we check ourselves
  ignore-unfixed: true  # Skip CVEs with no fix available (can't action them)
  vuln-type: "os,library"  # Scan OS packages AND library packages
```

Key attribute: `ignore-unfixed: true` — if a CVE exists but no patched version is available, there's nothing a developer can do. Failing on unfixable CVEs creates noise and developer fatigue without any security benefit.

**Critical check step:**
```bash
if grep -qE "CRITICAL" "trivy-report.txt"; then
  echo "critical_found=true" >> $GITHUB_OUTPUT
fi
```
Parses the text report with `grep`. If the word `CRITICAL` appears anywhere, sets the output flag. The caller uses this for decision-making.

### Trivy scan scope: `vuln-type: "os,library"`

- `os` — scans OS-level packages: apt packages (libssl, libcurl, glibc, etc.)
- `library` — scans application-level packages: pip packages, npm packages in the image

This is broader than Snyk SCA which only checks your declared dependencies — Trivy checks every package installed in the final image, including transitive OS dependencies.

### Alternatives to Trivy

| Tool | Strengths | Weakness |
|---|---|---|
| **Trivy** ✅ | Free, fast, comprehensive, multi-format output | Requires internet for DB updates |
| **Grype** (Anchore) | Fast, open source | Smaller vulnerability DB than Trivy |
| **Snyk Container** | Same Snyk platform, actionable advice | Paid for full features |
| **Clair** | Open source, widely used | Complex setup, no CLI |
| **Docker Scout** | Native Docker Hub integration | Limited outside Docker ecosystem |
| **Amazon ECR scanning** | Built-in for ECR users | AWS-only |

---

## 6. ci-docker-publish.yml — Docker Build & Push

**File:** `.github/workflows/ci-docker-publish.yml`

### What problem does this solve?

After code passes all security gates (SAST, SCA, Trivy), it needs to be packaged as a Docker image and published to a registry so Kubernetes can pull and run it. This workflow handles that — with several safeguards to prevent accidents.

### Inputs

| Input | Purpose |
|---|---|
| `service-name` | Becomes part of image name: `lavenbloom-<service-name>` |
| `environment` | `develop` or `main` — controls tag format |
| `target-env` | `dev` or `prod` — used in tag prefix |
| `release-tag` | For production: the explicit semver tag (e.g., `v1.2.0`) from GitHub Release |

### Outputs

| Output | Example value | Purpose |
|---|---|---|
| `image-tag` | `dev-a3f9c12b...` or `v1.2.0` | Passed to `cd-template.yml` to update Helm values |
| `image-full` | `rnld101/lavenbloom-auth-service:dev-abc` | Full registry reference |

### Step-by-step logic with all guard rails

**Step 1: Checkout with `fetch-depth: 0`**

Full git history fetched so we can inspect existing git tags in the next step.

**Step 2: Generate image tag**
```bash
if [ "${{ inputs.environment }}" = "main" ]; then
  TAG="${{ inputs.release-tag }}"          # e.g. "v1.2.0" from GitHub Release
else
  TAG="${{ inputs.target-env }}-${GITHUB_SHA}"  # e.g. "dev-a3f9c12b..."
fi
IMAGE_NAME="lavenbloom-${{ inputs.service-name }}"
echo "tag=$TAG" >> $GITHUB_OUTPUT
echo "full_image=${{ secrets.DOCKER_USERNAME }}/$IMAGE_NAME:$TAG" >> $GITHUB_OUTPUT
```
- **Dev tags** include the full git SHA — this uniquely identifies exactly which commit produced this image. When a bug is found, you can immediately trace it to a commit.
- **Prod tags** use the release tag provided by the GitHub Release event — this is set by the developer creating the release in GitHub (e.g., `v1.2.0`).

**Step 3: Validate existing git tag**
```bash
if git rev-parse "$TAG" >/dev/null 2>&1; then
  TAG_COMMIT=$(git rev-list -n 1 "$TAG")
  CURRENT_COMMIT=$(git rev-parse HEAD)
  if [ "$TAG_COMMIT" != "$CURRENT_COMMIT" ]; then
    echo "❌ Tag exists but points to different commit!"
    exit 1
  fi
fi
```
**Guard rail 1:** If a git tag with this name already exists AND it points to a *different* commit, the workflow fails. This prevents accidentally re-tagging an old commit as a new version. A release tag must point to the exact commit being built.

**Step 4: Check if image already exists in Docker Hub**
```bash
if docker manifest inspect ${{ steps.tag.outputs.full_image }} > /dev/null 2>&1; then
  echo "exists=true" >> $GITHUB_OUTPUT
fi
```
**Guard rail 2:** `docker manifest inspect` calls the Docker Hub API to check if an image with this exact tag already exists. If yes, the build and push steps are skipped. This prevents:
- Rebuilding the same image (waste of CI minutes)
- Overwriting an already-published image (dangerous for prod images)

**Step 5 & 6: Build and Push (conditional)**
```yaml
- name: Build Docker Image
  if: steps.image_check.outputs.exists != 'true'
  run: docker build -t ${{ steps.tag.outputs.full_image }} .

- name: Push Docker Image
  if: steps.docker_build.outputs.built == 'true'
  run: docker push ${{ steps.tag.outputs.full_image }}
```
The push only happens if the build happened. The build only happens if the image doesn't already exist. Two-layer conditional prevents partial states.

### Commented-out sections explained

The workflow has two commented-out blocks:

**Commented block 1: Semantic versioning via git tags**
```yaml
# uses: mathieudutour/github-tag-action@v6.2
```
This action would automatically calculate the next semantic version (e.g., `v1.2.3`) based on commit messages. **Currently not used** because the project chose to use GitHub Releases instead — the developer manually specifies the version when creating a release. The auto-calculation approach is an alternative for teams that want fully automated versioning.

**Commented block 2: Automatic git tag creation**
```yaml
# git tag ${{ steps.tag.outputs.tag }}
# git push origin ${{ steps.tag.outputs.tag }}
```
This would create a git tag in the service repo pointing to the built commit. **Currently not used** — the workflow relies on the GitHub Release event triggering with a pre-existing tag. Uncommenting this enables fully automated git tagging.

---

## 7. cd-template.yml — GitOps Helm Update

**File:** `.github/workflows/cd-template.yml`

### What is GitOps and why does this matter?

**GitOps** is the practice of using a Git repository as the single source of truth for infrastructure state. Instead of running `kubectl set image ...` to deploy, you commit a change to a Git file, and a tool (ArgoCD) detects the change and applies it to the cluster.

**Why this is better than imperative deployment:**
- Every deployment is a git commit — you have a full audit log of who deployed what and when
- Rollback = `git revert` — simple, auditable, reliable
- No humans need cluster access to deploy — reduces attack surface
- The cluster state is always traceable to a specific git commit

### Inputs explained

| Input | Example | Purpose |
|---|---|---|
| `service-name` | `auth-service` | Helm chart folder name in the charts repo |
| `service-key` | `authService` | Root YAML key in values file (camelCase) |
| `image-tag` | `dev-a3f9c12b` | The new tag written into `image.tag` |
| `environment` | `develop` | Which branch of the charts repo to push to |
| `target-env` | `dev` | Selects `values-dev.yaml` or `values-prod.yaml` |
| `charts-repo` | `lavenbloom/lavenbloom-charts` | Which repo to clone and modify |
| `charts-path` | `microservices` | Path prefix within the charts repo |

### Step-by-step logic

**Step 1: Clone the charts repo (not the service repo)**
```yaml
- uses: actions/checkout@v4
  with:
    repository: ${{ inputs.charts-repo }}   # lavenbloom/lavenbloom-charts
    token: ${{ secrets.HELM_REPO_PAT }}     # PAT with push access
    path: charts                            # Clone into ./charts/ subfolder
    ref: ${{ inputs.environment }}          # develop or main branch
```
The workflow clones a *different* repository (the charts repo) using a Personal Access Token. Without the PAT, it would only have read access. The service repo itself is NOT cloned in this workflow — we only need the charts repo.

**Step 2: Install yq**
```yaml
- uses: mikefarah/yq@v4.44.1
```
`yq` is a YAML processor (like `jq` but for YAML). It can read and write YAML values from the command line. Pinned to `v4.44.1` for reproducibility — not `@master` which would pull the latest unpinned version.

**Step 3: Update the image tag in values file**
```bash
FILE="charts/microservices/auth-service/values-dev.yaml"
yq -i '."authService".image.tag = "dev-a3f9c12b"' "$FILE"
```
`yq -i` means "in-place" — modify the file on disk. The expression `."authService".image.tag` navigates the YAML structure:
```yaml
authService:        # service-key
  image:
    tag: "dev-abc"  # ← this value is replaced
```

**Step 4: Validate YAML**
```bash
yq eval '.' "charts/microservices/auth-service/values-dev.yaml" > /dev/null
```
`yq eval '.'` re-parses the file. If it produces an error, it means the file is malformed YAML. This is a safety net — if the yq write step corrupted the file (e.g., wrong quoting), this catches it before committing broken YAML.

**Step 5: Commit and push**
```bash
cd charts
git config user.name "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"
git add "microservices/auth-service/values-dev.yaml"

if git diff --staged --quiet; then
  echo "⏭️ No changes detected"
else
  git commit -m "cd(auth-service): update image tag to dev-abc in values-dev.yaml [develop]"
  git push origin develop
fi
```
- `git diff --staged --quiet` — checks if there are any staged changes. If the tag is already the same value, there's nothing to commit. This prevents empty commits cluttering the git log.
- The commit message follows **Conventional Commits** format: `cd(scope): description` — enables automated changelog generation.
- The commit is pushed to the charts repo, which ArgoCD monitors. ArgoCD polls every 3 minutes and applies the change.

### The GitOps delivery chain in full

```
Developer merges PR to develop
        ↓
ci-docker-publish.yml builds image
        ↓  outputs: image-tag=dev-abc123
cd-template.yml clones charts repo
        ↓
Updates values-dev.yaml: tag: "dev-abc123"
        ↓  git push to charts repo/develop
ArgoCD polls charts repo every 3 min
        ↓
ArgoCD detects new commit
        ↓
ArgoCD runs: helm upgrade auth-service
        ↓
Kubernetes pulls new image and rolls out
        ↓
Deployment complete — all automated
```

---

## 8. ci-notify.yml — Failure Alerts

**File:** `.github/workflows/ci-notify.yml`

### Logic

Only runs when at least one job result is `"failure"`. Sends an email via SendGrid API containing:
- Which jobs failed and which succeeded
- Branch, actor, commit SHA, commit message
- Direct link to the failed Actions run

### Key design

Called with `if: always()` in the caller so it runs even when other jobs fail. It inspects the `needs.<job>.result` values passed as inputs. If all are `"success"` or `"skipped"`, the notification step exits early without sending email — no noise on green builds.

### Alternatives to SendGrid for notifications

| Tool | Channel |
|---|---|
| **SendGrid** ✅ | Email |
| **Slack Actions** | Slack channel message |
| **PagerDuty** | On-call alerting |
| **Microsoft Teams** | Teams channel webhook |
| **GitHub native** | PR status checks only (no email) |

---

## 9. Full Pipeline Flow

```
Pull Request opened
    │
    ├─▶ [sast]    SonarQube quality gate
    │       ↓
    ├─▶ [sca]     Snyk dependency CVE check
    │       ↓
    ├─▶ [trivy]   Container image CVE scan (temp build, never pushed)
    │       ↓
    └─▶ [pr-check] Aggregated gate — required by branch protection

Push to develop (after PR merged)
    │
    ├─▶ [dev-publish]  Build + push image tagged dev-{SHA}
    │       ↓
    └─▶ [dev-cd]       Update values-dev.yaml → ArgoCD syncs dev cluster

GitHub Release created (manually by developer/lead)
    │
    ├─▶ [publish]  Build + push image tagged v1.x.x
    │       ↓
    └─▶ [cd]       Update values-prod.yaml → ArgoCD syncs prod cluster
```

---

## 10. Environment Management

Two completely separate clusters — not just namespaces. Separation at cluster level means:
- A mistake in dev cannot affect prod
- Different resource quotas per cluster
- Different ArgoCD instances monitoring different branches

| Aspect | Dev | Prod |
|---|---|---|
| Trigger | Push to `develop` | GitHub Release |
| Image tag | `dev-{full SHA}` | `v1.x.x` (semver) |
| Values file updated | `values-dev.yaml` | `values-prod.yaml` |
| Charts branch | `develop` | `main` |
| ArgoCD monitors | `develop` branch | `main` branch |

---

## 11. How a DevOps Engineer Releases a New Version

### Day-to-day development flow

1. Developer creates feature branch from `develop`
2. Opens PR → pipeline runs SAST + SCA + Trivy + pr-check
3. PR is reviewed and merged to `develop`
4. Pipeline automatically builds `dev-{SHA}` image and deploys to dev cluster
5. QA tests on dev environment

### Releasing to production

1. Lead/DevOps engineer creates a **GitHub Release** in the service repo:
   - Goes to: `Releases → Draft a new release`
   - Tag: `v1.2.0` (semantic versioning: major.minor.patch)
   - Description: changelog
   - Publish release
2. GitHub fires `release.types: [created]` event
3. `ci-docker-publish.yml` builds the image tagged `v1.2.0`, pushes to Docker Hub
4. `cd-template.yml` updates `values-prod.yaml` in the charts repo with `tag: "v1.2.0"`
5. ArgoCD detects the commit on `main` branch of charts repo
6. ArgoCD runs `helm upgrade`, Kubernetes rolls out the new version
7. Monitor with `kubectl rollout status deployment/auth-service -n backend`

### Rollback procedure

```bash
# Option 1: Git revert (preferred — keeps audit trail)
git revert <commit-sha-that-updated-values-prod>
git push origin main
# ArgoCD detects and rolls back automatically

# Option 2: Manual helm rollback
helm rollback auth-service 1 -n backend

# Option 3: Force ArgoCD to previous sync
argocd app rollback auth-service
```

---

## 12. Q&A

**Q: Why does the SAST workflow use `fetch-depth: 0` but others don't?**
A: Only SonarQube needs the full git history — for blame data and incremental analysis. Other tools only need the current file state, so shallow clones (`fetch-depth: 1`, the default) are faster and sufficient.

**Q: Why is the Snyk step `|| true` but the Trivy step is `exit-code: "0"`?**
A: Same intent, different syntax. Snyk is a CLI tool — `|| true` tells the shell to ignore its exit code. Trivy is a GitHub Action — `exit-code: "0"` is an action parameter that tells the action not to fail the step. Both approaches suppress automatic failure so we can upload the report first and do our own check.

**Q: What happens if `cd-template.yml` pushes to the charts repo but ArgoCD is down?**
A: The commit still lands in the charts repo. When ArgoCD comes back online, it polls and detects the unsynced commit, then applies it. The delivery is eventually consistent — GitOps is resilient to temporary ArgoCD downtime.

**Q: Why use a PAT (`HELM_REPO_PAT`) instead of the `GITHUB_TOKEN`?**
A: `GITHUB_TOKEN` is scoped to the repository the workflow runs in. It cannot push to a *different* repository (`lavenbloom-charts`). A PAT with `repo` scope on the charts repo can. In enterprise settings, a GitHub App token is preferred over a PAT because it can be scoped to specific repos and auto-rotates.

**Q: What is the difference between `SAST`, `SCA`, and container scanning?**
A: They cover different layers:
- **SAST** — your *source code* (what you wrote)
- **SCA** — your *declared dependencies* (what you imported)
- **Container scanning** — the *full image filesystem* (everything in the Docker image, including OS packages your base image installed)

**Q: Could the three security scans run in parallel?**
A: Yes, technically. They are currently sequential (`sca needs sast`). Running them in parallel would make the PR pipeline faster. The sequential order was chosen to save CI minutes on fast-failing — if SAST fails immediately, there's no point running SCA.

**Q: What is a "quality gate" in SonarQube?**
A: A configurable set of conditions: e.g., "new code coverage must be above 80%", "no new CRITICAL security issues", "code duplication below 3%". The quality gate is defined on the SonarQube server. If any condition fails, the gate fails, blocking the merge. This is separate from the "scan" — the scan uploads data; the gate evaluates it.

**Q: Why is `snyk-to-html` installed globally (`npm install -g`) even for Python services?**
A: `snyk-to-html` is a Node.js tool that converts Snyk JSON output to HTML. It runs after the Snyk scan regardless of the service's runtime. The runner always has Node.js available, so `npm install -g` works even on Python service pipelines.

**Q: What does `docker manifest inspect` do exactly?**
A: It queries the Docker Hub API (without downloading the image) to check if a manifest (image metadata) exists for a given tag. It returns exit code 0 if the tag exists, non-zero if not. This is used as a lightweight existence check before deciding whether to build.
