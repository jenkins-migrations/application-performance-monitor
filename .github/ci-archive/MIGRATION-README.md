# Jenkins to GitHub Actions migration

## Migration status

The scripted Jenkins pipeline in `Jenkinsfile` was migrated to
`.github/workflows/ci.yml`. The original configuration is retained at
`.github/ci-archive/Jenkinsfile`.

## Preserved behavior

| Jenkins behavior | GitHub Actions equivalent |
| --- | --- |
| Checkout from SCM | `actions/checkout` |
| External workspace shared by the `linux` and `test` nodes | One GitHub-hosted job with a workspace shared by all sequential steps |
| `mvn clean install -DskipTests` | Build step |
| `mvn package -DskipTests` | Package step |
| Archive and fingerprint `target/*.jar` | Upload the JAR as the `package` workflow artifact |
| `mvn test` and Surefire XML publication | Unit-test step and `unit-test-results` artifact |
| `mvn failsafe:integration-test` and Failsafe XML publication | `mvn failsafe:integration-test failsafe:verify` and the `integration-test-results` artifact |
| Mark the build failed when a test command fails | Default GitHub Actions command failure handling |
| Clean the external workspace after tests | Ephemeral GitHub-hosted runner cleanup |

The Jenkins External Workspace Manager and test-results plugin calls were
expanded directly into workflow workspace and artifact operations; there are
no unresolved shared-library calls.

The added Failsafe `verify` goal checks the integration-test results and fails
the GitHub Actions job when integration tests fail.

## Triggers and runner

The workflow runs for pushes, pull requests, and manual dispatches. Jenkins did
not declare triggers in the archived file, so these standard repository events
replace Jenkins job-level SCM configuration. Jenkins node labels `linux` and
`test` are consolidated onto `ubuntu-latest` to preserve the shared workspace
without requiring self-hosted runners.

## Secrets and variables

No Jenkins credentials or secrets were used, so no GitHub Actions secrets are
required.

`JAVA_VERSION` is an optional GitHub Actions repository variable. It defaults
to `17`; set it to the JDK version formerly installed on the Jenkins agents if
that differs.

## Security and maintenance

The workflow grants only read access to repository contents. Marketplace
actions are maintained by GitHub and pinned to the immutable commits for
`actions/checkout` v7.0.1, `actions/setup-java` v6.0.0, and
`actions/upload-artifact` v7.0.1.

## Validation results

- `actionlint` v1.7.7 completed successfully for `.github/workflows/ci.yml`
  after adding the Failsafe `verify` goal.
- The workflow YAML parsed successfully with PyYAML.
- The pinned action commit SHAs were verified against their documented release
  tags.
- A complete workflow execution was not possible because the repository
  contained only `README.md` and the Jenkins configuration at migration time;
  the Maven project files, including `pom.xml`, are not present.
- The archived Jenkins file is byte-for-byte identical to the original tracked
  file.

The internal migration knowledge base was not accessible from the available
`jenkins-migrations/.github-private` repository path during this migration.
