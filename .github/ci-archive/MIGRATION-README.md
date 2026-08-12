# Jenkins → GitHub Actions Migration Report

This directory archives the original Jenkins pipeline definitions that were migrated to
GitHub Actions. The archived files are kept for reference only and are **no longer used**.

## 1. Migration Summary

| Original Jenkins file | Pipeline type | New GitHub Actions workflow |
| --- | --- | --- |
| `Jenkinsfile` (repo root) | Declarative | `.github/workflows/ci.yml` |
| `multinodeparallel/Jenkinsfile` | Scripted (Groovy) | `.github/workflows/multi-node-parallel.yml` |
| `complexdeployment/Jenkinsfile` | Declarative | `.github/workflows/complex-deployment.yml` |

Archived copies:

- `.github/ci-archive/Jenkinsfile`
- `.github/ci-archive/multinodeparallel-Jenkinsfile`
- `.github/ci-archive/complexdeployment-Jenkinsfile`

No Jenkins shared libraries (`@Library`, `vars/` functions) were referenced by these
pipelines, so no inline shared-library expansion was required.

## 2. Conversion Details

### 2.1 `Jenkinsfile` → `ci.yml`

| Jenkins | GitHub Actions |
| --- | --- |
| `agent any` | `runs-on: ubuntu-latest` |
| `parameters { booleanParam / string / choice }` | `on.workflow_dispatch.inputs` (`boolean`, `string`, `choice`) |
| `triggers { cron('@daily') }` | `on.schedule: - cron: '0 0 * * *'` |
| `triggers { pollSCM('H/5 * * * *') }` | Native `on.push` / `on.pull_request` events (no polling needed) |
| `options { disableConcurrentBuilds() }` | `concurrency.group` with `cancel-in-progress: false` |
| `options { timeout(time: 10, unit: 'MINUTES') }` | `timeout-minutes: 10` |
| `options { skipDefaultCheckout(true) }` | Explicit `actions/checkout` step only where needed |
| `options { timestamps() }` | Built-in — GitHub Actions logs are always timestamped |
| `options { buildDiscarder(logRotator(numToKeepStr:'5')) }` | Not expressible in workflow YAML; use repository *Settings → Actions → Artifact and log retention* |
| `checkout scm` | `actions/checkout` |
| `echo` / `sh` | `run:` |

Parameter defaults are re-applied with shell parameter expansion (`${ENVIRONMENT:-dev}`)
because `inputs.*` are empty for `schedule`, `push` and `pull_request` runs.

### 2.2 `multinodeparallel/Jenkinsfile` → `multi-node-parallel.yml`

| Jenkins | GitHub Actions |
| --- | --- |
| Groovy `builders` map + `parallel builders` | `strategy.matrix` (each leg runs concurrently) |
| `node('precise')` | `runs-on: ubuntu-22.04` |
| `node('trusty')` | `runs-on: ubuntu-latest` |
| `node('windows')` | `runs-on: windows-latest` |
| `isUnix()` branching (`sh` vs `bat`) | Two steps guarded by `if: matrix.unix` / `if: ${{ !matrix.unix }}`, `shell: cmd` for Windows |
| `stage('Checkout') { checkout scm }` | `actions/checkout` |
| `archiveArtifacts artifacts: 'build/**/*', fingerprint: true` | `actions/upload-artifact` (one artifact per matrix leg) |

> **Note:** Ubuntu `precise` (12.04) and `trusty` (14.04) are end-of-life and have no
> GitHub-hosted equivalent; the closest supported images are used instead.

### 2.3 `complexdeployment/Jenkinsfile` → `complex-deployment.yml`

| Jenkins stage / step | GitHub Actions |
| --- | --- |
| `environment { ... }` | Workflow-level `env:` |
| `parameters { choice / booleanParam / string }` | `workflow_dispatch.inputs` |
| `stage('Preparation')` | `preparation` job; `env.BUILD_TAG` / `env.IMAGE_TAG` become job `outputs` |
| `env.BUILD_NUMBER` | `github.run_number` (`$GITHUB_RUN_NUMBER`) |
| `currentBuild.description` | `$GITHUB_STEP_SUMMARY` |
| `stage('Build & Test') { parallel { ... } }` | Three independent jobs: `unit-tests`, `security-scan`, `code-quality` |
| `publishTestResults` / `publishCoverage` / `publishHTML` | `actions/upload-artifact` |
| `withSonarQubeEnv` + `waitForQualityGate()` | `SonarSource/sonarqube-scan-action` with `-Dsonar.qualitygate.wait=true` and `timeout-minutes: 10` |
| `docker.build` / `docker.withRegistry` / `image.push` | `docker/setup-buildx-action`, `docker/login-action`, `docker/build-push-action` |
| `sh 'trivy image ...'` | `aquasecurity/trivy-action` |
| `stage('Deploy')` + `try/catch` rollback | `deploy` job; the try/catch is expressed as an `if helm upgrade ... ; then ... fi` block that runs `helm rollback` when `ROLLBACK_ON_FAILURE` is true |
| `stage('Health Check')` with `retry(5)` | `health-check` job with a bash retry loop |
| `when { not { expression { target == 'production' } } }` | `if: ${{ inputs.DEPLOYMENT_TARGET != 'production' }}` |
| `stage('Production Approval')` with `input` | `production-approval` job bound to the `production-approval` GitHub Environment (required reviewers), `timeout-minutes: 30` |
| `stage('Smoke Tests')` with `parallel(...)` | `smoke-tests` job with a 3-leg matrix (API / UI / Performance) |
| `post { success / failure }` + `slackSend` | `post` job with `if: always()` and `slackapi/slack-github-action` steps guarded by `contains(needs.*.result, 'failure')` |
| `post { always { archiveArtifacts ... } }` | `actions/download-artifact` + `actions/upload-artifact` in the `post` job |
| `cleanWs()` | Not needed — every job runs on a fresh runner |

## 3. Required Secrets and Variables

Configure these under **Settings → Secrets and variables → Actions** before running
`complex-deployment.yml`:

| Secret | Jenkins equivalent | Purpose |
| --- | --- | --- |
| `DOCKER_REGISTRY_USERNAME` | `docker-registry-credentials` (username/password credential) | Registry login |
| `DOCKER_REGISTRY_PASSWORD` | `docker-registry-credentials` (username/password credential) | Registry login |
| `SONAR_TOKEN` | `withSonarQubeEnv('SonarQube')` server config | SonarQube authentication |
| `SONAR_HOST_URL` | `withSonarQubeEnv('SonarQube')` server config | SonarQube server URL |
| `KUBE_CONFIG` | Jenkins agent kubeconfig | Base64-encoded kubeconfig for `helm`/`kubectl` |
| `SLACK_BOT_TOKEN` | Slack Notification plugin | `slackSend` replacement |

Non-secret configuration (`APP_NAME`, `DOCKER_REGISTRY`, `HELM_CHART`, `SLACK_CHANNEL`)
is kept in the workflow-level `env:` block, mirroring the Jenkins `environment` block.

Required GitHub Environments:

- `dev`, `staging`, `production` — used by the `deploy` job.
- `production-approval` — configure **required reviewers** to reproduce the Jenkins
  `input` approval gate.

## 4. Validation

All workflows were validated with [`actionlint`](https://github.com/rhysd/actionlint):

```
$ actionlint
$ echo $?
0
```

All third-party actions are pinned to a full commit SHA with the human-readable version
in a trailing comment:

| Action | Version |
| --- | --- |
| `actions/checkout` | `v7.0.1` |
| `actions/upload-artifact` | `v7.0.1` |
| `actions/download-artifact` | `v8.0.1` |
| `actions/setup-java` | `v5.7.0` |
| `docker/setup-buildx-action` | `v4.2.0` |
| `docker/login-action` | `v4.6.0` |
| `docker/build-push-action` | `v7.3.0` |
| `aquasecurity/trivy-action` | `v0.36.0` |
| `SonarSource/sonarqube-scan-action` | `v8.2.1` |
| `azure/setup-helm` | `v5.0.1` |
| `slackapi/slack-github-action` | `v4.0.0` |

## 5. Manual Follow-up

1. Add the secrets listed in section 3 and create the four environments.
2. Set artifact/log retention in repository settings to replace `buildDiscarder`.
3. Provide a `Dockerfile` and the Helm chart referenced by `complex-deployment.yml`, plus
   the `make`/`build.bat` targets used by `multi-node-parallel.yml`; the Jenkins pipelines
   assumed these existed on the agents.
4. Install/provide `dependency-check`, `newman`, `cypress` and `artillery`, which were
   pre-installed Jenkins agent tooling.
5. Review the deployment ordering: the original Jenkins pipeline performed the production
   approval *after* deploying, and that behaviour was preserved verbatim. Consider moving
   the approval gate before `deploy` for production targets.
