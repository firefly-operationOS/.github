# `.github` — Shared workflows for firefly-operationOS

This repository hosts the reusable GitHub Actions workflows consumed by every repo
in the `firefly-operationOS` organization. Pattern modeled on
[`fireflyplatform/.github`](https://github.com/fireflyplatform/.github).

## Workflows

| File | Purpose | Trigger in caller |
|------|---------|-------------------|
| `java-ci.yml` | Build a Maven project (default goal `verify`). Configures `~/.m2/settings.xml` with the GitHub Packages servers and uploads surefire/failsafe reports. | `pull_request`, `push` (non-deploy paths) |
| `build-image.yml` | Build the Spring Boot OCI image of a `-web` module via `spring-boot:build-image` and push it to `ghcr.io/firefly-operationos/<artifact>` (tags `<version>` and `latest`). | `push` to `main` (only on services with a `-web` module) |
| `java-release.yml` | Deploy Maven artifacts to GitHub Packages and create/update a GitHub Release tagged `v<version>`. Tolerates 409 (artifact already exists). | `push` to `main`, or `workflow_dispatch` |

All three accept `inputs.java-version` (default `25`).

## Calling them from a service repo

A typical caller (`.github/workflows/ci.yml` in a service repo) looks like:

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  workflow_dispatch:

jobs:
  build:
    uses: firefly-operationOS/.github/.github/workflows/java-ci.yml@main
    secrets: inherit
```

For services that publish a container image:

```yaml
  image:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: [build]
    uses: firefly-operationOS/.github/.github/workflows/build-image.yml@main
    with:
      web-module: core-idp-catalog-web
    secrets: inherit
```

## Required org/repo secrets

These workflows expect the following secrets to be available via
`secrets: inherit` from the caller:

| Secret | Used for |
|--------|----------|
| `PACKAGES_TOKEN` | Read/write `read:packages`/`write:packages` on `firefly-operationOS` (Maven settings + GHCR). A PAT or fine-grained token works. |
| `GITHUB_TOKEN` | Injected by GitHub. Used by `docker/login-action` (GHCR push) and `gh release create` in `java-release.yml`. |

`PACKAGES_TOKEN` is typically defined at the **org** level so every repo using
`secrets: inherit` picks it up automatically.

## Maven repositories assumed in `settings.xml`

The reusable workflows generate a `~/.m2/settings.xml` that activates the
`github-packages` profile, with two repositories:

- `https://maven.pkg.github.com/firefly-operationOS/idp-backend-parent` — the
  org's parent POM repo, used as the URL anchor; GitHub Packages resolves every
  artifact in the org through this single URL.
- `https://maven.pkg.github.com/fireflyframework/fireflyframework-parent` — the
  external Firefly Framework parent + BOM artifacts.

If a new external Maven source is needed across the org, add a `<server>` and a
`<repository>` here in all three workflows.
