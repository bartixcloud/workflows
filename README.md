# bartixcloud workflows

Reusable GitHub Actions workflows for the `bartixcloud` org. Public repository:
nothing tenant-specific, secret or internal belongs in here.

| Workflow | Purpose |
| --- | --- |
| `.github/workflows/ci.yml` | one required `ci` check: lint, format, type check, tests |
| `.github/workflows/build.yml` | build the repo image, push to ghcr.io, expose its digest |

Consume them by tag (`@v1`); breaking changes get a new major tag.

## ci

`ci` runs as a single job named `ci`, so one required status check on `main`
covers the whole baseline.

What it runs for `language: python`: `uv sync --locked --all-groups`, then
`ruff check`, `ruff format --check`, `pytest` (only when a `tests/` directory
exists) and `ty check` (only when `uv.lock` contains `name = "ty"`). Optional
steps self-skip, so a repo without `ty` or tests stays green without extra
inputs. `python` is the only implemented language; `node` follows later, and any
other value yields a green job that runs no language steps.

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `runs-on` | yes | — | JSON array of runner labels |
| `language` | yes | — | `python` |
| `python-version` | no | empty → `.python-version` | Python version for setup-uv |

Caller:

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
jobs:
  ci:
    uses: bartixcloud/workflows/.github/workflows/ci.yml@v1
    with:
      runs-on: '["self-hosted","<tenant>","build"]'
      language: python
```

## build

Builds and pushes `ghcr.io/<owner>/<repo>[/<image>]`, lower-cased, tagged
`sha-<sha>` and `latest`. It outputs `digest` (bare `sha256:...`) and `ref`
(`ghcr.io/...@sha256:...`), and writes the same pin line to the job summary.

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `runs-on` | yes | — | JSON array of runner labels |
| `context` | no | `.` | build context directory |
| `dockerfile` | no | `Dockerfile` | Dockerfile path, relative to `context` |
| `image` | no | empty | sub-name: publish `ghcr.io/<owner>/<repo>/<image>` |

The called workflow requests `packages: write`, but a reusable workflow can only
narrow the caller's token, so the calling job must grant it.

Caller, on push to `main` plus manual dispatch:

```yaml
name: build
on:
  push:
    branches: [main]
  workflow_dispatch:
jobs:
  build:
    permissions:
      contents: read
      packages: write
    uses: bartixcloud/workflows/.github/workflows/build.yml@v1
    with:
      runs-on: '["self-hosted","<tenant>","build"]'
```

The calling job exposes the workflow outputs as its own, so a later job can read
them via `needs`:

```yaml
jobs:
  publish:
    needs: build
    runs-on: ["self-hosted","<tenant>","build"]
    steps:
      - run: echo "${{ needs.build.outputs.ref }}"
```

The `ref` output and the "Image digest — pin this" summary line are the same immutable reference.

## Runner labels

Every caller passes its own JSON label array:

```yaml
runs-on: '["self-hosted","<tenant>","build"]'
```

`<tenant>` scopes the queue to that tenant's build VM, `build` is the tier.
Runner group `build-ephemeral` serves private repos only. There is no fallback:
a caller on the wrong labels queues forever instead of failing, so a stuck job
means wrong labels. GitHub-hosted runners are avoided because they burn free
minutes — hence the self-hosted convention.

## Release model

This repo builds; it never deploys. There is deliberately no deploy workflow
here and none is planned.

Deploying is a separate act in `<tenant>/infra`: pin the immutable digest
(`ghcr.io/<owner>/<repo>@sha256:...`) and cut a signed tag there. The tenant's
convergence loop applies whatever digest that tag points at. `latest` is a
convenience pointer and is never a deployment pin.

Registry: once zot exists, the registry moves from `ghcr.io/<owner>/<repo>` to
`registry.bartix.de/<tenant>/...`; only host and path prefix change.
