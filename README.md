# shared-github-actions

Repository to store workflows that we'll re-use in other PNC repositories.

These workflows are designed for Maven-based Java projects and can be included
in other repositories using the `workflow_call` mechanism.

## Available Workflows

### Maven CI (`maven-ci.yml`)
Standard Continuous Integration workflow for Maven projects that we use to test
PRs. For some of the workflows, we can also further customize it by specifying
the Java version etc.

- **Tasks**: Checkout code, set up Java (default: 21), run build command (`mvn
  clean install`), and check for code formatting errors.


### Maven Release (`maven-release.yml`)
Workflow for performing a release to Maven Central (Sonatype).

- **Tasks**: Configures Git, sets up Java and GPG, performs `release:prepare`
  and `release:perform`, and pushes changes/tags back to the repository.


### Maven Snapshot (`maven-snapshot.yml`)
Workflow for deploying snapshot versions to Maven Central.

- **Tasks**: Deploy our SNAPSHOT version of our project to Maven Central
  Optionally builds and pushes a Quarkus Jib image to Quay.io.


### Maven Set Version (`maven-set-version.yml`)
Workflow to update the version in a Maven `pom.xml`.
- **Tasks**: Updates the version using `versions:set` and commits/pushes the change.


### Maven Jib (`maven-jib.yml`)
*Work in Progress* workflow to build Jib images and upload to a container
registry.


### Validate GitHub Action (`validate-gh-action.yml`)
Workflow used to lint and validate GitHub Actions using `actionlint` and
`zizmor`. This is used in this repository's `validate.yml` to run for PRs.

## Usage Example

To use one of these workflows in your repository, add a new workflow file
(e.g., `.github/workflows/ci.yml`) and reference the shared workflow:

```yaml
name: Run PR test

on:
  pull_request:
    branches: [ * ]

jobs:
  build:
    uses: project-ncl/shared-github-actions/.github/workflows/maven-ci.yml@<sha> # <tag>
    with:
      # if you want to customize the java_version
      java_version: '17'
```

A Github repository example using those workflows can be found
(here)[https://github.com/project-ncl/environment-driver/tree/main/.github/workflows]
