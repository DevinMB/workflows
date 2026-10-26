# workflows

Reusable GitHub Actions workflows for DevinMB's apps.

## app-ci.yml

Build, test, scan and publish a container image, and merge Dependabot updates that pass.

- **Pull requests:** build the image (with layer caching), run the tests inside it, start it and wait for its `HEALTHCHECK`, and scan it with Trivy (fixable HIGH and CRITICAL fail the build).
- **Dependabot PRs:** after those gates pass, patch and minor updates are merged and the main build is started; majors get the `waits-for-devin` label.
- **main:** the same gates, then push `ghcr.io/<owner>/<app>:sha-<7>` and `:main`.

```yaml
# .github/workflows/ci.yml in the app repo
name: ci
on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: write       # the gate merges Dependabot PRs
  pull-requests: write  # ...and labels majors
  packages: write       # push to GHCR on main
  actions: write        # the gate starts the main build after merging
jobs:
  ci:
    uses: DevinMB/workflows/.github/workflows/app-ci.yml@main
    with:
      image: ghcr.io/devinmb/<app>
      test-command: python -m pytest -q
```

| Input | Default | |
| --- | --- | --- |
| `image` | required | GHCR image name |
| `context`, `dockerfile` | `.`, `Dockerfile` | build context and Dockerfile (relative to the context) |
| `test-command` | empty | run inside the built image with `sh -c`; empty skips tests |
| `smoke` | `true` | start the image and wait for its `HEALTHCHECK` to report healthy |
| `smoke-env` | empty | `KEY=value` lines for the smoke container (never secrets) |
| `caller-workflow` | `ci.yml` | the caller's file, dispatched on main after a gate merge |
| `gate` | `true` | run the Dependabot gate after this build (see below for several images) |

### Several images in one repo

Call `app-ci.yml` once per image with `gate: false`, then the gate once, after all of them:

```yaml
jobs:
  api:
    uses: DevinMB/workflows/.github/workflows/app-ci.yml@main
    with: { image: ghcr.io/devinmb/<app>-api, context: api, gate: false }
  web:
    uses: DevinMB/workflows/.github/workflows/app-ci.yml@main
    with: { image: ghcr.io/devinmb/<app>-web, context: web, gate: false }
  gate:
    needs: [api, web]
    uses: DevinMB/workflows/.github/workflows/dependabot-gate.yml@main
```

All actions are pinned to commit SHAs. Inputs reach shell steps through environment variables, never by expression interpolation inside a script.

