# Jenkins to GitHub Actions migration report

Issue: #3  
Migration date: 2026-08-26

## Scope and inventory

All Jenkins definitions in the repository were analyzed and archived. No shared
library references or related scripts were present.

| Archived source | Pipeline type | Replacement |
| --- | --- | --- |
| `root/Jenkinsfile` | Declarative; parameters, SCM polling, daily cron, options | `../workflows/parameters-and-triggers.yml` |
| `complexdeployment/Jenkinsfile` | Declarative; parallel validation, image build, Helm deployment, rollback, tests, notifications | `../workflows/complex-deployment.yml` |
| `multinodeparallel/Jenkinsfile` | Scripted; dynamic parallel node map | `../workflows/multi-platform-build.yml` |

The original paths were deleted after their contents were moved here. Archived
files remain available for audit and rollback but are not executable by
Jenkins from their former locations.

## Behavior and mapping

### Parameters and triggers

- Jenkins `booleanParam`, `string`, and `choice` parameters map to typed
  `workflow_dispatch` inputs.
- `cron('@daily')` maps to `0 0 * * *` in UTC.
- Jenkins SCM polling maps to GitHub's native `push` event; GitHub receives
  repository events directly and does not need five-minute polling.
- `disableConcurrentBuilds()` maps to a workflow-level concurrency group.
- The 10-minute timeout maps to `timeout-minutes: 10`.
- `buildDiscarder(numToKeepStr: '5')` has no log-count equivalent. GitHub
  repository retention policy governs logs; generated artifacts use a
  five-day retention period where applicable.
- `skipDefaultCheckout(true)` plus explicit `checkout scm` maps to one explicit
  pinned checkout step.

### Multi-platform build

- The scripted Groovy map and `parallel builders` map to one matrix job with
  `fail-fast: false`.
- EOL Jenkins labels `precise` and `trusty` map to supported Linux runner
  images `ubuntu-22.04` and `ubuntu-24.04`; `windows` maps to
  `windows-latest`.
- `isUnix()` branches map to `runner.os` conditions. Unix retains `make` and
  `make test`; Windows retains `build.bat` and `test.bat`.
- Jenkins artifact archiving/fingerprinting maps to uniquely named workflow
  artifacts. GitHub artifacts are content-addressed; no separate fingerprint
  switch is needed.
- The source had no Jenkins trigger, so the replacement is manual-only.

### Complex deployment

- Jenkins parameters map to typed manual inputs. Deployment concurrency is
  isolated by target environment.
- Preparation emits validated job outputs for the short commit/build tag and
  full image name. User input is passed through environment variables rather
  than interpolated into scripts.
- Unit tests, dependency scanning, and SonarQube analysis remain parallel
  jobs. The Sonar invocation waits up to ten minutes for its quality gate.
- Maven packaging, Docker build/push, the `latest` tag, and the non-blocking
  HIGH/CRITICAL Trivy scan preserve source behavior.
- Helm deploy and Kubernetes readiness checks are retained. A shell error trap
  performs `helm rollback ... 0` when requested and a prior release exists.
- GitHub Environments replace Jenkins `input`. Configure required reviewers on
  the `production` environment; approval now occurs before deployment rather
  than after it, avoiding an unapproved production change.
- Health checks retry five times at ten-second intervals. Integration tests
  skip production. API, UI, and performance smoke tests run as a fail-fast-off
  matrix.
- Jenkins test/report and artifact publishers map to pinned artifact uploads.
  Slack notification uses a webhook directly, avoiding an additional action.
- Jenkins workspace cleanup is unnecessary on ephemeral hosted runners.

## Triggers and runners

| Workflow | Triggers | Runners |
| --- | --- | --- |
| Parameters and triggers | Push, daily UTC schedule, manual | `ubuntu-latest` |
| Multi-platform build | Manual | `ubuntu-22.04`, `ubuntu-24.04`, `windows-latest` |
| Complex deployment | Manual | `ubuntu-latest` |

The complex workflow is intentionally manual because it publishes and deploys.
Configure the `dev`, `staging`, and `production` GitHub Environments. Add
required reviewers to `production`.

## Secrets and variables

Create secrets at the repository or environment scope. Prefer environment
scope for deployment credentials.

| Name | Type | Jenkins source | Purpose |
| --- | --- | --- | --- |
| `REGISTRY_USERNAME` | Secret | Username half of `docker-registry-credentials` | Container registry login |
| `REGISTRY_PASSWORD` | Secret | Password/token half of `docker-registry-credentials` | Container registry login through stdin |
| `SONAR_HOST_URL` | Secret | `withSonarQubeEnv('SonarQube')` | SonarQube server URL |
| `SONAR_TOKEN` | Secret | `withSonarQubeEnv('SonarQube')` | SonarQube authentication |
| `KUBE_CONFIG_B64` | Secret | Jenkins agent cluster configuration | Base64-encoded kubeconfig, written mode `0600` to runner temp storage |
| `SLACK_WEBHOOK_URL` | Secret | Jenkins Slack plugin configuration | Optional result notification |
| `APP_NAME` | Variable | `microservice-app` | Optional application-name override |
| `DOCKER_REGISTRY` | Variable | `registry.example.com` | Optional registry-host override |
| `HELM_CHART` | Variable | `microservice-chart` | Optional chart reference override |
| `SLACK_CHANNEL` | Variable | `#deployments` | Documented source channel; webhook controls delivery |

No credential values were present in the Jenkinsfiles, and none were copied
into the workflows.

## Runner and repository prerequisites

The source repository currently contains pipeline examples but no application
sources, `pom.xml`, Dockerfile, Makefile, Windows batch files, Helm chart,
Postman/Cypress/Artillery assets, or test configuration. Add those files before
running the corresponding workflows.

GitHub-hosted runners must have, or the repository must install with verified
versions and checksums, the source pipeline's external tools:

- `dependency-check`, `trivy`, `helm`, `kubectl`, and `jq`
- `newman`, `cypress`, and `artillery`
- Maven/Java, Docker, Make, and the Windows batch scripts

Java is configured with the pinned official `actions/setup-java` action.
Avoid mutable `curl | sh`, unpinned package downloads, and unpinned container
images when adding tool installation.

## Security decisions

- Workflow permissions are reduced to read-only repository contents.
- All actions are maintained by GitHub and pinned to full commit SHAs:
  - `actions/checkout` v7.0.1:
    `3d3c42e5aac5ba805825da76410c181273ba90b1`
  - `actions/upload-artifact` v7.0.1:
    `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a`
  - `actions/setup-java` v6.0.0:
    `dd06d9cba3e5552c54d9f8ea23572deb30010f7c`
- Registry credentials use `docker login --password-stdin`.
- The kubeconfig is decoded only into temporary runner storage with restrictive
  permissions.
- HTTPS calls require TLS 1.2 or later. Dynamic tags are validated before use.
- Secrets are read only by the steps that require them.

## Validation

- `actionlint` v1.7.7 passed for all three workflows with no findings.
- `git diff --check` passed.
- Archive byte comparisons confirmed each moved Jenkinsfile is identical to
  its original Git version, and no Jenkinsfile remains outside the archive.
- Every `uses:` reference was checked as a full 40-character commit SHA.
- Secret scanning passed for all seven created/archive files with no findings.
- CodeQL's GitHub Actions analysis completed with zero alerts.
- Action dependency advisory lookup should be repeated in an authenticated
  environment; the migration environment could not query advisories because
  its GitHub token was unavailable.
- Runtime execution was not possible because the application files, external
  tools, credentials, environments, and deployment infrastructure are absent.

## Assumptions and differences

- GitHub push events are the intended replacement for Jenkins SCM polling.
- UTC midnight is acceptable for Jenkins `@daily`, whose exact execution time
  Jenkins normally hashes.
- Supported hosted runner versions replace unsupported Ubuntu Precise/Trusty.
- Production protection is configured in GitHub Environment settings.
- `example.com` endpoints and registry/chart defaults are placeholders carried
  forward from the source.
- A deployment rollback restores Helm revision `0`, matching the source; the
  recorded application version is used only to decide whether rollback exists.
- Slack notification is optional and skipped when no webhook secret exists.

## Rollback

1. Disable or remove the three files under `.github/workflows/`.
2. Move each archived Jenkinsfile back to its original path:
   - `root/Jenkinsfile` to `/Jenkinsfile`
   - `complexdeployment/Jenkinsfile` to `/complexdeployment/Jenkinsfile`
   - `multinodeparallel/Jenkinsfile` to `/multinodeparallel/Jenkinsfile`
3. Re-enable the corresponding Jenkins jobs and credential bindings.
4. Remove GitHub secrets/environments only after Jenkins execution is verified.
5. Preserve this report until the rollback and audit window closes.
