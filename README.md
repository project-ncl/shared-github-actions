
# Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Available Workflows](#available-workflows)
  - [Maven CI (`maven-ci.yml`)](#maven-ci-maven-ciyml)
  - [Gradle CI (`gradle-ci.yml`)](#gradle-ci-gradle-ciyml)
  - [SBT CI (`sbt-ci.yml`)](#sbt-ci-sbt-ciyml)
  - [NPM CI (`npm-ci.yml`)](#npm-ci-npm-ciyml)
  - [Mend CI (`mend-ci.yml`)](#mend-ci-mend-ciyml)
  - [Maven Release (`maven-release.yml`)](#maven-release-maven-releaseyml)
  - [Maven Snapshot (`maven-snapshot.yml`)](#maven-snapshot-maven-snapshotyml)
  - [Gradle Snapshot (`gradle-snapshot.yml`)](#gradle-snapshot-gradle-snapshotyml)
  - [SBT Snapshot (`sbt-snapshot.yml`)](#sbt-snapshot-sbt-snapshotyml)
  - [Maven Set Version (`maven-set-version.yml`)](#maven-set-version-maven-set-versionyml)
  - [Maven Jib (`maven-jib.yml`)](#maven-jib-maven-jibyml)
  - [Validate GitHub Action (`validate-gh-action.yml`)](#validate-github-action-validate-gh-actionyml)
  - [Usage Example](#usage-example)
- [Available Actions](#available-actions)
  - [Maven Build (`maven-build/action.yml`)](#maven-build-maven-buildactionyml)
  - [Gradle Build (`gradle-build/action.yml`)](#gradle-build-gradle-buildactionyml)
  - [NPM Build (`npm-build/action.yml`)](#npm-build-npm-buildactionyml)
  - [SBT Build (`sbt-build/action.yml`)](#sbt-build-sbt-buildactionyml)
  - [Mend (`mend/action.yml`)](#mend-mendactionyml)
- [Misc](#misc)
  - [Dependabot](#dependabot)
  - [GitHub Releases](#github-releases)
  - [GitHub Action validations](#github-action-validations)

<!-- END doctoc -->

# Introduction

This repository is used to store GitHub Action workflows (and other stuff) that we'll re-use in other PNC repositories

These workflows are designed for Maven-based Java projects and can be included
in other repositories using the `workflow_call` mechanism.

# Available Workflows

## Maven CI (`maven-ci.yml`)
Standard Continuous Integration workflow for Maven projects that we use to test
PRs. For some of the workflows, we can also further customize it by specifying
the Java version etc. It is possible to use this within a matrix job.

- **Tasks**: Checkout code, set up Java, set up Maven, run build command, check for code formatting errors, and optionally push build artifact (which is used by Maven Mend workflow).
- **Inputs**: The following inputs are available to be overridden
  * java_version (default: `21`)
  * build_command (default: `mvn -B -V clean install`)
  * fetch_all_commits (default: `false`)
  * maven_version (default: `3.9.15`)
  * upload_artifacts (default: `false`)
- **Permissions**: `contents: read`

<details>
<summary>Here is an example of using this in a matrix job</summary>

```yaml
    strategy:
      fail-fast: false
      matrix:
        maven-version: [3.3.9, 3.5.4, 3.6.3, 3.8.7, 3.9.0, 3.9.8]
        java-version: ['11']
        # Use include to add a specific Maven/JDK combination
        include:
          - maven-version: 3.9.11
            java-version: '17'
          - maven-version: 3.9.15
            java-version: '17'
    uses: project-ncl/shared-github-actions/.github/workflows/maven-ci.yml@ef31e53140344cac0421b7211cc0940182409106 # v0.0.16
    with:
      build_command: "mvn -B -V clean install -Prun-its"
      java_version: ${{ matrix.java-version }}
      maven_version: ${{ matrix.maven-version }}
```
</details>

## Gradle CI (`gradle-ci.yml`)
Standard CI workflow for Gradle projects.

- **Tasks**: Checkout code, set up Java, set up Gradle, run build command, optionally run coverage tests, and optionally push build artifact (which is used by Mend workflow). Can also run optional pre-build script.
- **Inputs**: The following inputs are available to be overridden
  * java_version (default: `17`)
  * gradle_version (default `wrapper`)
  * build_args
  * pre_build_script
  * fetch_all_commits (default: `false`)
  * upload_coverage (default: `false`)
  * coverage_files (report files to upload)
  * upload_artifacts (default: `false`)
- **Permissions**: `contents: read`

## SBT CI (`sbt-ci.yml`)
Standard CI workflow for Scala/SBT projects.

- **Tasks**: Checkout code, set up Java, set up SBT, run build command, and optionally push build artifact (which is used by Mend workflow).
- **Inputs**: The following inputs are available to be overridden
  * java_version (default: `8`)
  * build_args
  * resolvers (contents of SBT resolves file)
  * fetch_all_commits (default: `false`)
  * upload_artifacts (default: `false`)
- **Permissions**: `contents: read`

## NPM CI (`npm-ci.yml`)
Standard CI workflow for NPM projects.

- **Tasks**: Checkout code, set up Node.js, set up NPM, run build command, optionally run tests, optionally check for code formatting errors, and optionally push build artifact (which is used by Mend workflow).
- **Permissions**: `contents: read`

## Mend CI (`mend-ci.yml`)
Workflow to run Mend analysis, both SCA (Software Composition Analysiss) and SAST (Static Application Security Testing), on Maven projects. Because it has to have access to secrets in the organization or repository, on Pull Requests, it is meant to run `on: workflow_run`.

It depends on some build workflow with `upload_artifact` set to `true`, and before analysis, downloads an build artifact.

If `pr_feedback` is `false`, it downloads the build artifact, and runs the Mend analysis. This is meant for cronjob schedule, and push to main workflow runs.

Workflows run in the context of the PR from a fork do not have access to the secrets. It is meant to be executed `on: workflow_run` - this way, the Mend workflow is run in the context of the base repository. `pr_feedback` should be set to `true` and `triggering_run_id` set to ID of a triggering workflow. It downloads saved PR metadata and build artifact, sets check-run on the PR, runs the Mend analysis, and posts PR comment.

To exclude paths from being scanned, specify `SCA_EXCLUDES` and/or `SAST_EXCLUDES` values, in the comma-separated paths format: `'dir1/**,dir2/file.txt'`.

<details>
<summary>Example of push to main and cronjob usage</summary>

```yaml
name: Mend CLI scan for Maven

on:
  push:
    branches:
      - main
  schedule:
    - cron: "0 22 * * 0"

permissions:
  contents: read
  actions: read
  checks: write
  pull-requests: write
  security-events: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
      - uses: project-ncl/shared-github-actions/.github/actions/maven-build@<sha>
      - uses: actions/upload-artifact@<sha>
        with:
          name: pr-build
          path: .

  call-maven-mend-ci:
    needs: build
    uses: project-ncl/shared-github-actions/.github/workflows/mend-ci.yml@<sha>
    with:
      SCA: true
      SAST: true
    secrets:
      MEND_URL: ${{ secrets.MEND_URL }}
      MEND_USER_KEY: ${{ secrets.MEND_USER_KEY }}
      MEND_EMAIL: ${{ secrets.MEND_EMAIL }}
      MEND_ORGNAME: ${{ secrets.MEND_ORGNAME }}
      MEND_PRODUCTNAME: ${{ secrets.MEND_PRODUCTNAME }}
```
</details>

<details>
<summary>Example of PR usage</summary>

```yaml
name: Java CI with Maven

on:
  pull_request:

permissions:
  contents: read

jobs:
  build:
    uses: project-ncl/shared-github-actions/.github/workflows/maven-ci.yml@<sha>
    with:
      upload_artifacts: true
```

```yaml
name: Mend CLI scan for Maven PR

on:
  workflow_run:
    workflows: ["Java CI with Maven"]
    types: [completed]

permissions:
  contents: read
  actions: read
  checks: write
  pull-requests: write
  security-events: write

concurrency:
  group: mend-scan-${{ github.event.workflow_run.pull_requests[0].number || github.event.workflow_run.head_sha }}
  cancel-in-progress: true

jobs:
  scan:
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'pull_request'
    uses: project-ncl/shared-github-actions/.github/workflows/mend-ci.yml@<sha>
    with:
      SCA: true
      SAST: true
      triggering_run_id: ${{ github.event.workflow_run.id }}
      pr_feedback: true
    secrets:
      MEND_URL: ${{ secrets.MEND_URL }}
      MEND_USER_KEY: ${{ secrets.MEND_USER_KEY }}
      MEND_EMAIL: ${{ secrets.MEND_EMAIL }}
      MEND_ORGNAME: ${{ secrets.MEND_ORGNAME }}
      MEND_PRODUCTNAME: ${{ secrets.MEND_PRODUCTNAME }}
```
</details>

## Maven Release (`maven-release.yml`)
Workflow for performing a release to Maven Central (Sonatype). This can be manually run by going to the GitHub Actions tab and selecting the workflow.

- **Tasks**: Configures Git, sets up Java and GPG, performs `release:prepare`
  and `release:perform`, and pushes changes/tags back to the repository. The
  next version is set by bumping the patch version by 1 and putting the
  `-SNAPSHOT` suffix.
- **Inputs**: The following inputs are available to be overridden
  * ref_to_release (default: `''`)
  * java_version (default: `21`)
  * release_command (default `mvn -B -V release:prepare release:perform -DlocalCheckout=true -DpushChanges=false`)
  * fetch_all_commits (default: `false`)
  * jboss_parent_override: This is used to override variables from the jboss-parent (default `-Dcentral.serverId=central-publisher -Dcentral.autoPublish=false -DreleaseProfile=central-release -DsignTag=false`)
- **Permissions**: `contents: write`

Note that the `jboss-parent` overrides the release-plugin `tagNameFormat` to use `@{project.version}`. To revert to the default format add the following to the calling projects properties: `<tagNameFormat>@{project.artifactId}-@{project.version}</tagNameFormat>`

## Maven Snapshot (`maven-snapshot.yml`)
Workflow for deploying snapshot versions to Maven Central. Gated to the upstream repo so forks don't attempt to publish

- **Tasks**: Deploy our SNAPSHOT version of our project to Maven Central
  Optionally builds and pushes a Quarkus Jib image to Quay.io.
- **Inputs**: The following inputs are available to be overridden
  * project_name :  **Must** be set by te caller.
  * java_version (default: `21`)
  * snapshot_deploy_command (default `mvn -B -V deploy`)
  * fetch_all_commits (default: `false`)
  * quarkus_jib_image ( default: `false`)
  * jboss_parent_override: This is used to override variables from the jboss-parent (default `-Dcentral.serverId=central-publisher -Dcentral.sonatype.url=https://central.sonatype.com/repository/maven-snapshots -Pcentral-release -Dgpg.skip`)
- **Permissions**: `none`

## Gradle Snapshot (`gradle-snapshot.yml`)

Workflow for deploying snapshot versions to Gradle Central, similar to its Maven counterpart. Gated to the upstream repo so forks don't attempt to publish.

## SBT Snapshot (`sbt-snapshot.yml`)

Workflow for deploying snapshot versions to Sonatype Nexus, similar to its Maven counterpart. Gated to the upstream repo so forks don't attempt to publish.

The workflow writes `~/.sbt/.credentials` for SBT to authenticate against the snapshot repository.

## Maven Set Version (`maven-set-version.yml`)
Workflow to update the version in a Maven `pom.xml`. This can be manually run by going to the GitHub Actions tab and selecting the workflow.

- **Tasks**: Updates the version using `versions:set` and commits/pushes the change.
- **Permissions**: `contents: write`

This workflow should be manually called and an example of that may be seen [here](https://github.com/project-ncl/environment-driver/blob/main/.github/workflows/maven-set-version.yml). Its recommended that `on.workflow_dispatch` is used so the user can enter the appropriate values e.g.
```
on:
  workflow_dispatch: # Manual trigger
    inputs:
      new_version:
        description: '[Required] Version to set'
        required: true
        type: string
...
jobs:
    call-maven-set-version-job:
      permissions:
        contents: write # needed to push commit and tag back to the repository
      uses: project-ncl/shared-github-actions/.github/workflows/maven-set-version.yml@504efc82df7d4f73e3bd2b67bec31bcd532ed9e4 # v0.0.14
      with:
        new_version: ${{ inputs.new_version }}
```


## Maven Jib (`maven-jib.yml`)
*Work in Progress* workflow to build Jib images and upload to a container
registry.


## Validate GitHub Action (`validate-gh-action.yml`)
Workflow used to lint and validate GitHub Actions using `actionlint` and
`zizmor`. This is used in this repository's `validate.yml` to run for PRs.

- **Permissions**: `none`

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

A GitHub repository example using those workflows can be found
(here)[https://github.com/project-ncl/environment-driver/tree/main/.github/workflows]

# Available Actions

## Maven Build (`maven-build/action.yml`)

Sets up Java, sets up Maven, and runs build command.

## Gradle Build (`gradle-build/action.yml`)

Sets up Java, sets up Gradle, and runs build command.

## NPM Build (`npm-build/action.yml`)

Sets up Node.js, sets up NPM, and runs build command.

## SBT Build (`sbt-build/action.yml`)

Sets up Java, sets up SBT, and runs build command.

## Mend (`mend/action.yml`)

Downloads and installs Mend CLI, runs Mend SCA scan (if enabled), and runs Mend SAST scan (if enabled). Publishes artifacts of results of SCA/SAST scans.

Also has a `TAGS` property to associate Mend scan run with some data (e.g. branch name, PR number, etc.) - which is then visible in the Mend UI.

# Misc

## Dependabot

A sample dependabot file in `.github/dependabot.yml` is available that is both used in this repository and may be copied to other ProjectNCL repositories.

## GitHub Releases
A sample `.github/release.yml` configuration file from the [GitHub documentation](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes#configuring-automatically-generated-release-notes) has been added that may be copied to other ProjectNCL repositories.

## GitHub Action validations
We use (zizmor)[https://docs.zizmor.sh/] and
[actionlint](https://github.com/rhysd/actionlint) to validate that our GitHub
Actions are secure.

One of the requirements is to explicitly define a permissions key to specify
the level of access that the injected `GITHUB_TOKEN` should have.

For workflows that do not require to `git push` any changes, we set an empty
permissions field:
```
permissions: {}
```

and for those that do:
```
permissions:
  contents: write
```
For example, `maven-release.yml` requires to `git push` changes back to the
repository and needs those permissions.

Note that the final workflow should also specify the same permissions as the
shared workflow being used.
