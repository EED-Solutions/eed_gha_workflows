# EED GitHub Actions Workflows

This repository is the central home for the EED reusable GitHub workflows and
their composite actions.

## Components

- Composite actions live under `.github/actions/`:
  - `pytests`
  - `pytests_cov`
  - `ruff`
- Reusable workflows live under `.github/workflows/`:
  - `callable_pytests.yml`
  - `callable_pytests_cov.yml`
  - `callable_publish_pypi.yml`
  - `callable_ruff.yml`

Consumer repositories should call the reusable workflows rather than copy their
implementation.

## Using the workflows in a consumer repository

Create one `.github/workflows/ci.yml` in the consumer repository. Trigger it on
pushes to `main`, pull requests targeting `main`, and manual dispatch. The
workflow should call the three reusable workflows at a released version:

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  ruff:
    uses: EED-Solutions/eed_gha_workflows/.github/workflows/callable_ruff.yml@1.0.0
  pytests:
    uses: EED-Solutions/eed_gha_workflows/.github/workflows/callable_pytests.yml@1.0.0
  pytests-cov:
    uses: EED-Solutions/eed_gha_workflows/.github/workflows/callable_pytests_cov.yml@1.0.0
    with:
      package_name: my_python_package
```

Replace `my_python_package` with the importable Python package name. Keep the
distribution name in `pyproject.toml` separate if it uses kebab-case.

Use a published stable version tag such as `1.0.0`, not `main`, for production
consumer workflows. The reusable workflows also reference the composite
actions at the same version tag. This keeps both layers on a known release.
Update consumers to a newer stable tag when you are ready to adopt that
release.

Use prerelease tags such as `1.0.0-rc.1` only for integration testing. Mark
them as prereleases on GitHub and do not recommend them for production
consumers. Publish the stable tag on the same commit only after the
Copier-generated canary repository has validated the candidate.

## Publishing a Python package

Generated package repositories receive a separate `publish.yml` workflow. It
calls `callable_publish_pypi.yml` when a GitHub Release is published:

- GitHub prereleases publish to TestPyPI.
- Stable GitHub Releases publish to PyPI.
- The release tag must exactly match the version in `pyproject.toml` (for
  example, tag `1.2.0` for project version `1.2.0`).

Before the first upload, configure a [Trusted Publisher](https://docs.pypi.org/trusted-publishers/adding-a-publisher/)
for the package on each index. For a package that does not exist yet, configure
a [pending publisher](https://docs.pypi.org/trusted-publishers/creating-a-project-through-oidc/);
the first successful upload creates the project. Use the generated repository,
the workflow filename `publish.yml`, and the matching GitHub environment:
`testpypi` on TestPyPI and `pypi` on PyPI. Trusted Publishing uses GitHub OIDC,
so no PyPI API token needs to be stored as a repository secret. Configure
both publishers if you plan to publish release candidates to TestPyPI and
stable versions to PyPI.

The reusable workflow builds the wheel and source distribution, then publishes
those build artifacts in a separate job. Since publishing uses a reusable
workflow, validate the OIDC publisher configuration with a canary package on
TestPyPI before relying on production publishing.

## CI for this repository

This repository's own `ci.yml` runs Ruff directly on pushes to `main`, pull
requests targeting `main`, and manual dispatch. It does not call the reusable
workflows or composite actions hosted here. That avoids making this
repository's CI depend on a release tag that may not exist yet while a new
version is being prepared.

The `pytests.yml` and `ruff.yml` workflows are manual-dispatch wrappers around
the reusable workflows. They are useful for checking a published release and
are not the repository's pull-request CI.

## Releasing and validating a version

1. Make and review changes to the composite actions and reusable workflows.
2. Confirm this repository's standalone Ruff CI passes.
3. Merge the changes to `main`.
4. Create an immutable prerelease tag on that `main` commit, for example
   `1.0.0-rc.1`. Do not move or reuse a published tag.
5. Test the candidate through the Copier template and its generated canary.
6. If the canary passes, create the stable tag (for example `1.0.0`) on the
   same shared-workflow commit. If fixes are needed, merge them and publish a
   new prerelease tag, such as `1.0.0-rc.2`.
7. Update the Copier template to use the stable tag and verify the stable
   reference in the canary before rolling it out to other consumers.

The template's maintainer guide documents the branch-based canary test and
the complete release and rollout sequence:
[eed_package_template/CONTRIBUTING.md](https://github.com/EED-Solutions/eed_package_template/blob/main/CONTRIBUTING.md).
