# CI/CD Integration

The CLI requires the `UIGRAPH_TOKEN` environment variable and a gateway URL.

Set the gateway URL with `UIGRAPH_GATEWAY_URL` or `--api-url` when the gateway is
self-hosted. On hosted UIGraph, pass `--enterprise` instead and the CLI resolves the URL
itself, so neither the variable nor the flag is needed. `--enterprise=DEV` targets the dev
environment. `--api-url` and `--enterprise` cannot be passed together, and a command with
none of the three exits with code `1`.

## Installing the CLI

The CLI is distributed from the `uigraph-oss/uigraph-cli` repository on GitHub, both as Go
source and as the `uigraph/uigraph-cli` Docker image.

### Docker

Prefer this in CI. The image ships the CLI already compiled, so the pipeline skips both the
Go toolchain setup and the build, and every run uses the same binary.

```bash
docker pull uigraph/uigraph-cli:latest

docker run --rm \
  -v "$PWD:/workspace" -w /workspace \
  -e UIGRAPH_TOKEN \
  -e UIGRAPH_GATEWAY_URL \
  uigraph/uigraph-cli:latest sync --dry-run
```

The CLI reads `.uigraph.yaml` and every artifact path relative to the working directory, so
the repository root must be mounted and `-w` must point at it. `-e NAME` with no value
forwards the variable from the host shell; both must already be exported.

Pin a released tag such as `uigraph/uigraph-cli:v1.4.0` when the pipeline needs reproducible
runs. `latest` moves.

`uigraph-cli release` reads the tag at `HEAD`, so the mounted directory needs its full `.git`
— clone with the complete history before mounting it.

### Go

```bash
go install github.com/uigraph-oss/uigraph-cli
export PATH="$PATH:$(go env GOPATH)/bin"
```

Use this for local runs on a machine that already has Go, or where Docker is unavailable.

## CLI Commands

```bash
uigraph-cli sync
uigraph-cli sync --config .uigraph.yaml
uigraph-cli sync --dry-run
uigraph-cli sync --verbose
uigraph-cli sync --api-url https://gateway.your-org.com
uigraph-cli sync --enterprise
uigraph-cli sync --enterprise=DEV --dry-run
```

`uigraph-cli release` records one release event on the service timeline and exits. It reads
the tag at `HEAD`, so it belongs in a tag-triggered job with the full git history
checked out — a shallow clone has no tags and the command fails.

```bash
uigraph-cli release
uigraph-cli release --version v1.4.0
uigraph-cli release --notes-file RELEASE_NOTES.md
uigraph-cli release --dry-run
uigraph-cli release --enterprise
```

Notes are resolved in order: `--notes`, `--notes-file`, the annotated tag body, the
matching changelog section, then the commit subjects since the previous tag.

This skill never runs `release` and never generates release notes. It only wires the job
into the pipeline when the user asks for CI/CD templates.

If the pipeline runs `uigraph-cli release`, omit `timeline.releases.changelogPath` from
`.uigraph.yaml`. Both sources key the event as `release:<version>` and overwrite each
other. See `references/cost-tags-and-timeline.md`.

## GitHub Actions

Runners already have Docker, so the image replaces the `setup-go` and `go install` steps
outright.

```yaml
name: UIGraph Sync

on:
  push:
    branches: [main, master]
    tags: ['v*']
  pull_request:
    branches: [main, master]

env:
  UIGRAPH_IMAGE: uigraph/uigraph-cli:latest

jobs:
  uigraph-sync:
    if: "!startsWith(github.ref, 'refs/tags/')"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Dry run on PR
        if: github.event_name == 'pull_request'
        env:
          UIGRAPH_TOKEN: ${{ secrets.UIGRAPH_TOKEN }}
          UIGRAPH_GATEWAY_URL: ${{ secrets.UIGRAPH_GATEWAY_URL }}
        run: |
          docker run --rm -v "$PWD:/workspace" -w /workspace \
            -e UIGRAPH_TOKEN -e UIGRAPH_GATEWAY_URL \
            "$UIGRAPH_IMAGE" sync --dry-run

      - name: Sync to UIGraph on push
        if: github.event_name == 'push'
        env:
          UIGRAPH_TOKEN: ${{ secrets.UIGRAPH_TOKEN }}
          UIGRAPH_GATEWAY_URL: ${{ secrets.UIGRAPH_GATEWAY_URL }}
        run: |
          docker run --rm -v "$PWD:/workspace" -w /workspace \
            -e UIGRAPH_TOKEN -e UIGRAPH_GATEWAY_URL \
            "$UIGRAPH_IMAGE" sync

  uigraph-release:
    if: startsWith(github.ref, 'refs/tags/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Record release
        env:
          UIGRAPH_TOKEN: ${{ secrets.UIGRAPH_TOKEN }}
          UIGRAPH_GATEWAY_URL: ${{ secrets.UIGRAPH_GATEWAY_URL }}
        run: |
          docker run --rm -v "$PWD:/workspace" -w /workspace \
            -e UIGRAPH_TOKEN -e UIGRAPH_GATEWAY_URL \
            "$UIGRAPH_IMAGE" release
```

`fetch-depth: 0` stays on the release job. The mount carries `.git` into the container, and
`release` fails without the tags.

To stay on the Go toolchain instead, swap each `docker run` back for `uigraph-cli` and add
the `actions/setup-go@v5` and `go install` steps ahead of it.

## GitLab CI

```yaml
uigraph-sync:
  stage: deploy
  image: golang:1.23
  script:
    - go install github.com/uigraph-oss/uigraph-cli
    - export PATH="$PATH:$(go env GOPATH)/bin"
    - uigraph-cli sync
  only:
    - main
    - master
  variables:
    UIGRAPH_TOKEN: $UIGRAPH_TOKEN
    UIGRAPH_GATEWAY_URL: $UIGRAPH_GATEWAY_URL
  tags:
    - docker

uigraph-sync-dry-run:
  stage: test
  image: golang:1.23
  script:
    - go install github.com/uigraph-oss/uigraph-cli
    - export PATH="$PATH:$(go env GOPATH)/bin"
    - uigraph-cli sync --dry-run
  only:
    - merge_requests
  variables:
    UIGRAPH_TOKEN: $UIGRAPH_TOKEN
    UIGRAPH_GATEWAY_URL: $UIGRAPH_GATEWAY_URL
  tags:
    - docker

uigraph-release:
  stage: deploy
  image: golang:1.23
  script:
    - go install github.com/uigraph-oss/uigraph-cli
    - export PATH="$PATH:$(go env GOPATH)/bin"
    - uigraph-cli release
  only:
    - tags
  variables:
    UIGRAPH_TOKEN: $UIGRAPH_TOKEN
    UIGRAPH_GATEWAY_URL: $UIGRAPH_GATEWAY_URL
    GIT_DEPTH: 0
  tags:
    - docker
```

## Bitbucket Pipelines

```yaml
image: golang:1.23

pipelines:
  default:
    - step:
        name: UIGraph Sync
        script:
          - go install github.com/uigraph-oss/uigraph-cli
          - export PATH="$PATH:$(go env GOPATH)/bin"
          - uigraph-cli sync
        variables:
          UIGRAPH_TOKEN: $UIGRAPH_TOKEN
          UIGRAPH_GATEWAY_URL: $UIGRAPH_GATEWAY_URL

  tags:
    'v*':
      - step:
          name: UIGraph Release
          clone:
            depth: full
          script:
            - go install github.com/uigraph-oss/uigraph-cli
            - export PATH="$PATH:$(go env GOPATH)/bin"
            - uigraph-cli release
          variables:
            UIGRAPH_TOKEN: $UIGRAPH_TOKEN
            UIGRAPH_GATEWAY_URL: $UIGRAPH_GATEWAY_URL
```

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `UIGRAPH_TOKEN` | yes | API token for gateway authentication |
| `UIGRAPH_GATEWAY_URL` | unless `--enterprise` or `--api-url` is passed | Gateway to sync to |

The templates below leave the gateway URL out of the job. Set `UIGRAPH_GATEWAY_URL` as a
pipeline variable alongside `UIGRAPH_TOKEN`, or add `--enterprise` to each `uigraph-cli`
command on hosted UIGraph.

## Token Scopes

The token needs one scope per surface the config writes. A missing scope returns 403 and
the CLI stops there with exit code `2` — later steps never run. Grant every scope the
config needs before the first real run.

| Scope | Needed for |
|-------|------------|
| `services:write` | Service record, APIs, dependencies, diagrams, databases, queries, test packs, test cases, docs |
| `maps:write` | Maps, frames, focal points, component links |
| `billing:write` | `costTags` |
| `timeline:write` | `timeline` events and `uigraph-cli release` |
| `mlstudio:write` | `ml` projects, models, experiments, runs |

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Config error, validation error, or missing token |
| `2` | Gateway sync error |
