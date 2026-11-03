# AGENTS.md

This repository contains **no application code**. It is a library of reusable
GitHub Actions — composite actions and `workflow_call` reusable workflows — that
ProjectNCL (PNC) repositories consume via `uses:`. "Building" here means editing
and validating YAML; there is nothing to compile or run locally.

## Layout

- `.github/actions/<name>/action.yml` — composite actions (`maven-build`,
  `gradle-build`, `npm-build`, `sbt-build`, `mend`). Each sets up a toolchain and
  runs a build command. These are the low-level building blocks.
- `.github/workflows/<name>.yml` — reusable workflows (`on: workflow_call`) that
  callers reference. CI (`*-ci.yml`), snapshot deploy (`*-snapshot.yml`), release
  (`*-release.yml`), plus `mend-ci.yml`, `maven-set-version.yml`,
  `maven-jib-image-updater.yml`, and `validate-gh-action.yml`.
- `.github/workflows/private-*.yml` — **this repo's own** CI/release plumbing.
  They are not reusable and must not be copied by callers (each is headed with a
  comment saying so): `private-validate.yml` (runs lint on PRs),
  `private-toc.yml` (regenerates the README table of contents via `doctoc`),
  `private-release.yml` (manual workflow that bumps the tag and cuts a GitHub
  Release — titled `"0 Release Job"` so it sorts to the top of the Actions list).
- `.github/dependabot.yml`, `.github/release.yml` — sample files that are both
  used here and meant to be copied into other PNC repos (ecosystems are
  commented out and must be uncommented per repo).
- `README.md` — the authoritative, user-facing reference for every workflow's
  inputs, secrets, permissions, and caller examples. Keep it in sync with any
  input/secret/behavior change. Its table of contents is auto-generated; do not
  hand-edit the TOC block.

## Validation (the only "build/test" step)

There is no test suite. Correctness is enforced by two linters, run by
`validate-gh-action.yml` (and triggered on PRs that touch workflows via
`private-validate.yml`):

- **actionlint** — GitHub Actions syntax/semantics.
- **zizmor** — security linting (https://docs.zizmor.sh/).

To check changes locally before pushing, run these against the YAML you edited:

```sh
actionlint
zizmor .github/
```

## Conventions that must be preserved

These are load-bearing patterns enforced by zizmor/actionlint and by how callers
depend on this repo. New or edited files must follow them:

- **Pin third-party actions to a full commit SHA**, with the human-readable
  version in a trailing comment, e.g.
  `uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1`.
  Dependabot (`github-actions` ecosystem) bumps these weekly.
- **Explicit `permissions`.** Every workflow declares the minimal `GITHUB_TOKEN`
  scope: `permissions: {}` when nothing is pushed, `contents: write` when it
  pushes commits/tags. A caller must grant the same permissions as the shared
  workflow it invokes.
- **`persist-credentials: false`** on every `actions/checkout`.
- **Never interpolate `${{ inputs.* }}` directly into a `run:` shell.** Pass the
  value through an `env:` var and reference the env var in the script (see
  `maven-build/action.yml` → `INPUTS_BUILD_COMMAND`). This prevents script
  injection and is enforced by zizmor.
- **Fork-gating for publishing workflows.** Snapshot/release workflows take a
  required `project_name` (`<org>/<project>`) input and guard the job with
  `if: github.repository == inputs.project_name` so forks cannot publish.
- **`concurrency` group** with `cancel-in-progress: true` on workflows, keyed on
  `github.workflow`/`github.ref` (plus version inputs for the matrix-able CI).
- **`fetch_all_commits` input** conventionally maps to
  `fetch-depth: ${{ inputs.fetch_all_commits == true && '0' || '1' }}`.

## Releasing this repo

Callers pin to this repo by SHA with a version-tag comment, so a release is just
a new tag + GitHub Release. Run the manual `private-release.yml` workflow from
the Actions tab (inputs: `ref_to_release`, `tag_bump` = patch/minor/major). It
computes the next tag and generates release notes; it does not build anything.

When a change alters a workflow's inputs, secrets, permissions, or behavior,
update `README.md` to match — it is the contract callers read.
