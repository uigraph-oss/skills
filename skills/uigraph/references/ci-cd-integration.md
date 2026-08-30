# CI/CD Integration

The CLI requires the `UIGRAPH_TOKEN` environment variable and a gateway URL.

Set the gateway URL with `UIGRAPH_GATEWAY_URL` or `--api-url` when the gateway is
self-hosted. On hosted UiGraph, pass `--enterprise` instead and the CLI resolves the URL
itself, so neither the variable nor the flag is needed. `--enterprise=DEV` targets the dev
environment. `--api-url` and `--enterprise` cannot be passed together, and a command with
none of the three exits with code `1`.

## Installing the CLI

The CLI is distributed from the `uigraph-oss/uigraph-cli` repository on GitHub. Install it with Go:

```bash
go install github.com/uigraph-oss/uigraph-cli
export PATH="$PATH:$(go env GOPATH)/bin"
```

For other installation methods, including the published Docker image, read the installation section of that repository's README before running any command.

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

```yaml
name: UiGraph Sync

on:
  push:
    branches: [main, master]
    tags: ['v*']
  pull_request:
    branches: [main, master]

jobs:
  uigraph-sync:
    if: "!startsWith(github.ref, 'refs/tags/')"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: stable

      - name: Install UiGraph CLI
        run: |
          go install github.com/uigraph-oss/uigraph-cli
          echo "$(go env GOPATH)/bin" >> "$GITHUB_PATH"

      - name: Dry run on PR
        if: github.event_name == 'pull_request'
        env:
          UIGRAPH_TOKEN: ${{ secrets.UIGRAPH_TOKEN }}
          UIGRAPH_GATEWAY_URL: ${{ secrets.UIGRAPH_GATEWAY_URL }}
        run: uigraph-cli sync --dry-run

      - name: Sync to UiGraph on push
        if: github.event_name == 'push'
        env:
          UIGRAPH_TOKEN: ${{ secrets.UIGRAPH_TOKEN }}
          UIGRAPH_GATEWAY_URL: ${{ secrets.UIGRAPH_GATEWAY_URL }}
        run: uigraph-cli sync

  uigraph-release:
    if: startsWith(github.ref, 'refs/tags/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v5
        with:
          go-version: stable

      - name: Install UiGraph CLI
        run: |
          go install github.com/uigraph-oss/uigraph-cli
          echo "$(go env GOPATH)/bin" >> "$GITHUB_PATH"

      - name: Record release
        env:
          UIGRAPH_TOKEN: ${{ secrets.UIGRAPH_TOKEN }}
          UIGRAPH_GATEWAY_URL: ${{ secrets.UIGRAPH_GATEWAY_URL }}
        run: uigraph-cli release
```

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
        name: UiGraph Sync
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
          name: UiGraph Release
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
command on hosted UiGraph.

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
