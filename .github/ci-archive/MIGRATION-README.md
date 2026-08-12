# Jenkins-to-GitHub-Actions migration report

**Completed:** 2026-08-12
**Source configurations archived:** `Jenkinsfile`, `variablesharing/Jenkinsfile`

## Delivered workflows

| Archived Jenkins source | GitHub Actions workflow | Trigger |
| --- | --- | --- |
| `Jenkinsfile` | `.github/workflows/deploy-application.yml` | Manual `workflow_dispatch` |
| `variablesharing/Jenkinsfile` | `.github/workflows/variable-sharing.yml` | Manual `workflow_dispatch` |

No Jenkins trigger was declared in either source, so the workflows are manual rather
than inventing push or pull-request behavior.

## Main pipeline mapping

| Jenkins behavior | GitHub Actions equivalent |
| --- | --- |
| `agent any` | `ubuntu-latest` runner |
| `ENVIRONMENT`, `SKIP_TESTS`, `VERSION` parameters | Typed manual-dispatch inputs |
| Build and `mvn clean package -DskipTests=...` | `build` job with the same command |
| Conditional test stage | `test` job runs only when `skip_tests` is false |
| `publishTestResults` | Surefire XML is retained as a GitHub Actions artifact; GitHub does not natively convert it to Jenkins-style test UI publishing |
| Development, staging, and production `when` conditions | One deployment job selects the matching Kubernetes manifest path |
| Jenkins staging and production `input` approvals | GitHub Environments named `staging` and `production`; configure required reviewers and wait timers in repository settings |
| Production `DEPLOYMENT_TYPE` input | Required `deployment_type` choice input (`blue-green`, `rolling`, or `canary`) |
| Staging/production smoke test | Same `curl --fail` command after the corresponding deployment |
| `post { success/failure }` | `report-result` job runs with `always()` and reports the deployment outcome |

The GitHub Actions production strategy is selected when the workflow is dispatched,
before the protected-environment approval. Jenkins collected that choice during the
approval dialog. GitHub Environment approvals are the closest native, auditable
replacement for both approval steps.

## Variable-sharing pipeline mapping

`readMavenPom()` was replaced by Maven's `help:evaluate` expression for
`project.version`. `BUILD_RELEASE_VERSION`, `IS_SNAPSHOT`, and `GIT_TAG_COMMIT`
are calculated with equivalent Bash expressions. The Jenkins global `tags_extra`
assignment and its dependent stages execute in one Bash step, which intentionally
keeps the variable in the same process scope while preserving the three-stage
output and condition.

The checkout uses full history because `git describe --tags --always` needs tags.

## Secrets and environment configuration

Neither Jenkinsfile declares a Jenkins credential binding. The deployment commands
nevertheless require Kubernetes credentials that a Jenkins agent may previously
have supplied implicitly. For each GitHub Environment (`dev`, `staging`, and
`production`), create an environment secret named `KUBECONFIG_B64` containing the
base64-encoded kubeconfig. The workflow writes it to `~/.kube/config` with mode
`0600` only for the deployment job.

Also configure protected GitHub Environments:

- `staging`: required reviewers (and an optional five-minute wait timer).
- `production`: required reviewers (and an optional sixty-minute wait timer).

Ensure the runner can reach the Kubernetes API and the selected service health URL.
The workflow uses only SHA-pinned, GitHub-maintained actions:
`actions/checkout` (`11bd71901bbe5b1630ceea73d27597364c9af683`, v4.2.2) and
`actions/upload-artifact` (`ea165f8d65b6e75b540449e92b4886f43607fa02`, v4.6.2).
Maven and `kubectl` are expected from the Ubuntu hosted-runner image, matching the
original unspecified `agent any` tooling requirement.

## Validation and known blockers

- The workflow structure was reviewed against the two declarative Jenkins pipelines.
- `actionlint` was not installed in the migration environment, so syntax validation
  could not be run. Validate with `actionlint .github/workflows/*.yml` before the
  first production dispatch.
- The GitHub advisory lookup for the pinned Actions dependencies could not run
  because the migration environment has no GitHub token. Both actions are pinned
  to documented commit SHAs; re-run the advisory lookup in an authenticated
  environment before dispatching the workflow.
- This repository contains no Maven project, Kubernetes manifests, or test reports.
  Consequently, Maven packaging, deployment, smoke tests, and their environment
  connectivity could not be exercised locally.
- The requested migration prohibits commits, pushes, remote pull-request creation,
  and remote pull-request updates. Therefore no pull request was created or updated.

## Archive status

Both original Jenkins sources have been moved to this directory (the nested source
is named `variablesharing-Jenkinsfile` to avoid a filename collision) and removed
from their original paths. No shared-library calls were present, so none required
inline expansion.
