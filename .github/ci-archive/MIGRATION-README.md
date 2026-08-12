# Jenkins to GitHub Actions Migration Report

## Summary

The repository's Jenkins **scripted** pipeline (`Jenkinsfile`) has been migrated to a GitHub Actions
workflow at `.github/workflows/ci.yml`. The original Jenkins file has been archived in this
directory and removed from the repository root.

| Item | Value |
| --- | --- |
| Source system | Jenkins (scripted pipeline, Groovy) |
| Source file | `Jenkinsfile` (archived as `.github/ci-archive/Jenkinsfile`) |
| Target workflow | `.github/workflows/ci.yml` |
| Validation | `actionlint` 1.7.12 — passed with no findings |
| Shared libraries | None used (no `@Library` / `vars/` calls to expand) |

## Source pipeline analysis

The original pipeline used two `node` blocks sharing a Jenkins External Workspace
(`exwsAllocate 'diskpool1'` + `exws`):

1. `node('linux')` — stages `Checkout`, `Build` (`mvn clean install -DskipTests`),
   `Package` (`mvn package -DskipTests` + `archiveArtifacts target/*.jar`).
2. `node('test')` — stages `Unit Tests` (`mvn test` + publish surefire results) and
   `Integration Tests` (`mvn failsafe:integration-test` + publish failsafe results),
   wrapped in `try/catch/finally` with `cleanWs`.

## Conversion mapping

| Jenkins construct | GitHub Actions equivalent |
| --- | --- |
| `node('linux')` | `build` job with `runs-on: ubuntu-latest` |
| `node('test')` | `unit-tests` and `integration-tests` jobs with `runs-on: ubuntu-latest` |
| `stage('X')` | Named step (or job) within the workflow |
| `checkout scm` | `actions/checkout` |
| `sh 'mvn ...'` | `run: mvn --batch-mode ...` |
| `archiveArtifacts artifacts: 'target/*.jar', fingerprint: true` | `actions/upload-artifact` (`jars`) |
| `publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'` | `actions/upload-artifact` (`unit-test-results`, `if: always()`) |
| `publishTestResults testResultsPattern: 'target/failsafe-reports/*.xml'` | `actions/upload-artifact` (`integration-test-results`, `if: always()`) |
| `exwsAllocate 'diskpool1'` / `exws(...)` (shared workspace across nodes) | `build-output` artifact uploaded by `build` and downloaded by the test jobs |
| `try/catch { currentBuild.result = 'FAILURE'; throw e }` | Default GitHub Actions behavior — a failing step fails the job and the run |
| `finally { cleanWs ... }` | Not required — GitHub-hosted runners are ephemeral |
| No `triggers` block (build triggered externally) | `on: push` / `pull_request` (branch `main`) and `workflow_dispatch` |
| Implicit JDK/Maven tooling on the Jenkins agent | `actions/setup-java` (Temurin 17, Maven dependency cache) |

## Actions used (pinned to commit SHAs)

| Action | Version | SHA |
| --- | --- | --- |
| `actions/checkout` | v7.0.1 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |
| `actions/setup-java` | v5.7.0 | `b6effb05e454b25005698d916606bdc6ffcbf961` |
| `actions/upload-artifact` | v7.0.1 | `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` |
| `actions/download-artifact` | v8.0.1 | `3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c` |

All actions are published by the verified `actions` organization and pinned to release commit SHAs.

## Secrets, variables and credentials

| Type | Name | Notes |
| --- | --- | --- |
| Secrets | _none_ | The Jenkins pipeline contained no `withCredentials` blocks or credential bindings. |
| Variables | _none required_ | `JAVA_VERSION` and `JAVA_DISTRIBUTION` are defined as workflow-level `env` values. |

`GITHUB_TOKEN` is used implicitly with read-only `contents: read` permissions.

## Manual follow-up

- Confirm the Java version (`JAVA_VERSION`, currently `17`) and distribution match the project's
  requirements; adjust the workflow `env` block if needed.
- The Jenkins External Workspace shared a full workspace between agents; the workflow shares only
  the Maven `target/` directory. If other generated directories must be shared, extend the
  `build-output` artifact paths.
- If a rich test-report UI is desired instead of raw XML artifacts, add a test-reporting action
  that satisfies your organization's action allow-list.
- Adjust the `push` / `pull_request` branch filters if the default branch is not `main`.

## Validation

```
actionlint .github/workflows/ci.yml
# exit code 0 — no issues found
```
