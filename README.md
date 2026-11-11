# bartixcloud workflows

Reusable GitHub Actions workflows for the `bartixcloud` org and the orgs that
call it. Public repository: nothing tenant-specific, secret or internal belongs
in here.

| Workflow | Purpose |
| --- | --- |
| `.github/workflows/ci.yml` | one required `ci` check: lint, format, type check, tests (v1 entry point, kept for existing callers) |
| `.github/workflows/checks.yml` | run a caller-defined list of check commands with uv, pnpm or rust set up, optionally beside a throwaway Postgres |
| `.github/workflows/build.yml` | build the repo image, push to ghcr.io, expose its digest |
| `.github/workflows/release.yml` | no-PR release from conventional commits (version stamp, tag, GitHub release) |
| `.github/workflows/deploy.yml` | `nuk` deploys in three modes: `apply`, `preview`, `deploy` |

## Pin rule

Callers pin a workflow to the commit of a release tag, never to a branch:

```yaml
uses: bartixcloud/workflows/.github/workflows/<concern>.yml@<40-char SHA> # vX.Y.Z
```

The SHA is the commit the signed release tag `vX.Y.Z` points to (for an
annotated tag, the peeled commit, not the tag object). The trailing comment
keeps the version readable and lets Dependabot propose the next one. Breaking
changes get a new major tag; existing tags are never moved.

Third-party actions inside these workflows follow the same rule, and the org
policy in `policy/actions.json` (see below) enforces it.

## Runner labels

Every caller passes its own JSON label array:

```yaml
runs-on: '["self-hosted","<tenant>","build"]'
```

`<tenant>` scopes the queue to that tenant's build VM, `build` is the tier.
Runner group `build-ephemeral` serves private repos only. There is no fallback:
a caller on the wrong labels queues forever instead of failing, so a stuck job
means wrong labels. Public repos must use a GitHub-hosted label (for example
`'["ubuntu-latest"]'`), because self-hosted runners must not run fork PRs.

## Secrets

Secrets are mapped by name in the calling job. The called workflow declares the
names it accepts under `workflow_call.secrets`; the caller maps each one from
its own secret, whatever that is called in the caller's repo or org:

```yaml
jobs:
  deploy:
    uses: bartixcloud/workflows/.github/workflows/deploy.yml@<40-char SHA> # vX.Y.Z
    with:
      mode: deploy
      runs-on: '["self-hosted","<label>"]'
    secrets:
      APP_ENV: ${{ secrets.MY_DEPLOY_ENV }}
      SHARED_ENV: ${{ secrets.MY_SHARED_ENV }}
```

Do not use `secrets: inherit` across organizations. Name new secrets so that
they are not a personal access token (no `PAT` in the name, no `_TOKEN` suffix,
no `GH_` or `GITHUB_` prefix); the workflows here use the job's `GITHUB_TOKEN`.

## ci

`ci` runs as a single job named `ci`, so one required status check on `main`
covers the whole baseline.

What it runs for `language: python`: `uv sync --locked --all-groups`, then
`ruff check`, `ruff format --check`, `pytest` (only when a `tests/` directory
exists) and `ty check` (only when `uv.lock` contains `name = "ty"`). Optional
steps self-skip, so a repo without `ty` or tests stays green without extra
inputs. `python` is the only implemented language; any other value yields a
green job that runs no language steps.

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
    uses: bartixcloud/workflows/.github/workflows/ci.yml@<40-char SHA> # vX.Y.Z
    with:
      runs-on: '["self-hosted","<tenant>","build"]'
      language: python
```

## checks

`ci` is the baseline for a plain repo. `checks` is for a repo whose checks are its
own: a monorepo with a backend and a frontend, generated-artifact drift checks,
migrations against a real database, a Rust crate. The caller lists the
commands; the workflow sets up the toolchain, starts a throwaway Postgres when
asked, runs the commands with `bash -euo pipefail` (the first failure fails the
job) and removes the container afterwards. One call is one job, so a backend and
a frontend are two calls.

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `runs-on` | yes | — | JSON array of runner labels |
| `toolchain` | yes | — | `python` (uv), `node` (pnpm), `rust` or `none` |
| `commands` | yes, except `rust` | empty | script run in `working-directory`; for `rust` empty means `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, `cargo test --locked` |
| `working-directory` | no | `.` | where install and commands run |
| `python-version` | no | empty → `.python-version` | |
| `node-version` | no | empty → the runner image's node | |
| `install` | no | `uv sync --locked` / `pnpm install --frozen-lockfile` / `cargo fetch --locked` | |
| `env` | no | empty | extra `KEY=VALUE` lines for the commands |
| `postgres` | no | `false` | start Postgres; commands see `POSTGRES_PORT` (user `postgres`, password `test`, database `test`, `127.0.0.1` only) |
| `postgres-image` | no | `postgres:16-alpine` | |

`rust` uses the toolchain pinned in the shared runner image (nothing is
downloaded at job time), honours a repo `rust-toolchain.toml`, and caches the
cargo registry, git db and `target/` keyed on `Cargo.lock` and `rustc -V`.

The first step of `checks` and `build` prints `bcx-runner-image: <tag>` (the
infra tag of the runner image) or `bcx-runner-image: none` off the shared image.

The Postgres needs Docker on the runner; the `build` tier image ships it.

```yaml
jobs:
  backend:
    uses: bartixcloud/workflows/.github/workflows/checks.yml@<40-char SHA> # vX.Y.Z
    with:
      runs-on: '["self-hosted","<tenant>","build"]'
      toolchain: python
      working-directory: backend
      postgres: true
      commands: |
        uv run ruff check .
        uv run pytest
        DATABASE_URL="postgresql+asyncpg://postgres:test@127.0.0.1:${POSTGRES_PORT}/test" uv run alembic upgrade head
  crate:
    uses: bartixcloud/workflows/.github/workflows/checks.yml@<40-char SHA> # vX.Y.Z
    with:
      runs-on: '["self-hosted","<tenant>","build"]'
      toolchain: rust
```

## build

Builds and pushes `ghcr.io/<owner>/<repo>[/<image>]`, lower-cased, tagged
`sha-<sha>`, `latest` and any extra `tags`. It outputs `digest` (bare `sha256:...`) and `ref`
(`ghcr.io/...@sha256:...`), and writes the same pin line to the job summary.

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `runs-on` | yes | — | JSON array of runner labels |
| `context` | no | `.` | build context directory |
| `dockerfile` | no | `Dockerfile` | Dockerfile path, relative to `context` (joined as `<context>/<dockerfile>`) |
| `image` | no | empty | sub-name: publish `ghcr.io/<owner>/<repo>/<image>` |
| `ref` | no | empty → triggering commit | git ref to check out and build; `sha-<sha>` names that commit |
| `tags` | no | empty | extra image tags, newline-separated (e.g. `v1.2.3`) |

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
    uses: bartixcloud/workflows/.github/workflows/build.yml@<40-char SHA> # vX.Y.Z
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

## release

No release PR, no auto-merge, no personal access token. On push to the default
branch it derives the next version from conventional commits since the last
tag, stamps it into the in-code version files, commits that with `[skip ci]`
and creates the tag and the GitHub release. Outputs: `released` (`true` when a
version was released in this run) and `tag`.

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `runs-on` | yes | — | JSON array of runner labels; no default |
| `version-files` | no | empty | newline-separated files to stamp (`package.json`, `*.toml`, `*.py`) |
| `pre-1-breaking-minor` | no | `false` | while the last tag is 0.x, a breaking change bumps minor |

A tag or release created with `GITHUB_TOKEN` triggers no workflow, so a
release image is built in the same run, gated on the release job's output and
building the tag it just pushed:

```yaml
jobs:
  release:
    permissions:
      contents: write
    uses: bartixcloud/workflows/.github/workflows/release.yml@<40-char SHA> # vX.Y.Z
    with:
      runs-on: '["ubuntu-latest"]'
  image:
    needs: release
    if: needs.release.outputs.released == 'true'
    permissions:
      contents: read
      packages: write
    uses: bartixcloud/workflows/.github/workflows/build.yml@<40-char SHA> # vX.Y.Z
    with:
      runs-on: '["self-hosted","<tenant>","build"]'
      ref: ${{ needs.release.outputs.tag }}
      tags: ${{ needs.release.outputs.tag }}
```

## deploy

For hosts that run the `nuk` CLI. One job per `mode`:

| `mode` | What it does |
| --- | --- |
| `apply` | pull the images of a compose file, then `nuk apply` a manifest (images built by the app's own CI) |
| `deploy` | build one image from a Dockerfile, push it to GHCR, write the env files from secrets, then `nuk apply` |
| `preview` | per-PR ephemeral stack; `preview_action` is `deploy`, `teardown` or `cleanup` |

`runs-on` is required and has no default. The calling job must grant the
permissions of the chosen mode: `apply` needs `contents: read` and
`packages: read`, `deploy` needs `contents: read` and `packages: write`,
`preview` needs `contents: read`, `packages: read`, `pull-requests: write` and
`actions: read`. The secrets are mapped by name as shown under Secrets above.
The input table of `deploy.yml` is the reference; this section stays short on
purpose.

## Release model

Images are built here (`build`), versions are cut here (`release`), and the
`deploy` workflow applies an already pinned manifest on a host. Promoting a
deployment in a tenant's infra repository stays a separate act: pin the
immutable digest (`ghcr.io/<owner>/<repo>@sha256:...`) and cut a signed tag
there. `latest` is a convenience pointer and is never a deployment pin.

Registry: once zot exists, the registry moves from `ghcr.io/<owner>/<repo>` to
`registry.bartix.de/<tenant>/...`; only host and path prefix change.

## Policy and Dependabot

- `policy/actions.json` is the Actions policy the member orgs follow:
  `{"allowed_actions": "selected", "sha_pinning_required": true}`. The keys
  are the fields of the `actions/permissions` endpoint of the GitHub REST API,
  so the live org setting is compared key by key against this file.
- `.github/dependabot.yml` keeps this repo's own action pins current.
- `policy/dependabot-template.yml` is the starting point for a caller repo:
  `github-actions` weekly, plus the repo's own ecosystems, grouped, label
  `deps`. Copy it to `.github/dependabot.yml` and keep the ecosystems the repo
  uses. Dependabot reads the `# vX.Y.Z` comment next to each pinned SHA.

## Free-plan limits

Re-read from the GitHub documentation on 2026-11-11; check the pages again
before relying on a number.

- Included compute on the Free plan: 12 USD per month, expressed as a monetary
  amount, not minutes; per-minute rates apply (Linux 1-core x64 0.002 USD).
  Self-hosted runners do not use it. Artifact storage 500 MB (shared with
  GitHub Packages), cache 10 GB per repository.
  [About billing for GitHub Actions](https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions)
- Per job and run: 6 hours per job on GitHub-hosted runners, 35 days per
  workflow run, 20 concurrent jobs (5 macOS), 256 jobs per matrix, 50 re-runs
  per run, 500 KB per workflow file.
  [Actions limits](https://docs.github.com/en/actions/reference/limits)
- Reusable workflows: up to ten levels of nesting; one workflow file can call
  up to 50 unique reusable workflows, nested ones included.
  [Reusing workflow configurations](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)
