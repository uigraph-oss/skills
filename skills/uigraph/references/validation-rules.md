# Validation Rules

These are the rules the LLM/agent must manually verify by inspecting generated files. They also reflect constraints enforced by `config.Validate()` in the CLI.

After generating artifacts, validate the generated structure before finishing. Do this through LLM/agent file inspection and reasoning. If validation fails, fix only files generated during the current execution.

## Required Top-Level Fields

- `version` must be `1`
- `version` is required
- `service.name` must not be empty
- `service.category` must not be empty
- `service.description` must not be empty
- `service.repository.provider` must not be empty
- `service.repository.url` must not be empty
- `service.ownership.team` must not be empty when a `service` block is present

## Enums

| Field | Valid Values |
|-------|-------------|
| `service.repository.provider` | `github`, `gitlab`, `bitbucket` |
| `apis[*].type` | `openapi`, `graphql`, `grpc` |
| `dependencies[*].direction` | `upstream`, `downstream` (required) |
| `dependencies[*].type` | `http`, `graphql`, `grpc`, `database` (optional) |
| `dependencies[*].criticality` | `hard`, `soft` (required) |
| `architectureDiagrams[*]` | `name` and `path` required |
| `testPacks[*].type` | `smoke`, `regression`, `manual` |
| `testCases[*].type` | `api`, `manual` |
| `testCases[*].priority` | `p0`, `p1`, `p2`, `p3` |
| `databases[*].dialect` | `postgres`, `mysql`, `sqlite`, `dynamodb`, `mongodb`, `other` |
| `docs[*].fileType` | `pdf`, `html`, `markdown`, `doc`, `txt`, `image`, `video`, `audio`, `other` |
| `maps[*].frames[*].focalPoints[*].visibility` | `public`, `private` |
| `maps[*].frames[*].focalPoints[*].components[*].componentId` | `component_api-contract`, `component_test-case-suite`, `component_support-kb-troubleshooting`, `component_backend-flow-diagram` |
| `testCases[*].screenshots[*]` extension | `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.svg` |
| `ml[*].type` | `model`, `training` (required) |
| `ml[*].source.type` | `mlflow` (required) |
| `ml[*].models[*].problemType` | `classification`, `regression`, `ranking`, `generation`, `embedding`, `other` (optional) |

## File Existence Checks

All `path` fields must point to files that exist relative to `.uigraph.yaml`:

- `apis[*].path`
- `architectureDiagrams[*].path`
- `architectureDiagrams[*].contextPath`
- `databases[*].schemaPath`
- `docs[*].path`
- `maps[*].frames[*].imagePath`
- `queries[*].path` (when set instead of `queryText`)
- `queryFiles[*]` (each must be a YAML file with a `queries:` list)
- `testPacks[*].testCasesPath` (when set; a YAML file with a `testCases:` list)
- `testPacks[*].testCases[*].screenshots[*]` (must be a file, not a directory)
- `timeline.releases.changelogPath` (when set)

`timeline.decisions.paths` and `timeline.incidents.paths` are the exception: they are
glob patterns, and a pattern matching nothing is not an error. Verify by inspection that
each pattern actually matches files in the repository.

Generated artifact paths must stay under `.uigraph/`:

- `apis[*].path` must be under `.uigraph/openapi/` when generated.
- `architectureDiagrams[*].path` must be under `.uigraph/diagrams/` when generated.
- `architectureDiagrams[*].contextPath` must be under `.uigraph/diagrams/` when generated.
- `databases[*].schemaPath` must be under `.uigraph/db/`.
- `docs[*].path` must be under `.uigraph/docs/` when generated.
- `maps[*].frames[*].imagePath` must be under `.uigraph/maps/` when generated.
- `testPacks[*].testCases[*].screenshots[*]` must be under `.uigraph/tests/screenshots/` when generated.

## Repository URL Checks

- `service.repository.url` must come from the current git remote or an explicit user-provided URL.
- Do not copy placeholder repository URLs from templates.
- Normalize SSH GitHub/GitLab/Bitbucket remotes to HTTPS when possible.
- `service.repository.provider` must match the repository host.

## Database Schema File Checks

- Database artifact type must match discovered project evidence or explicit user input. Do not generate both SQL and NoSQL by default.
- SQL dialects (`postgres`, `mysql`, `sqlite`, `other` when SQL-like) must use `.sql` files under `.uigraph/db/`.
- NoSQL dialects (`dynamodb`, `mongodb`) must use `.json` files under `.uigraph/db/`.
- `databases[*].schemaPath` must point to a file extension that matches the declared dialect.

## Helper Script Checks

- Generated helper scripts must directly produce approved UiGraph artifacts.
- Generated helper scripts must be written only in JavaScript, Python, or Bash (`.sh`).
- Helper script language must match the project type: JavaScript for JavaScript projects, Python for Python projects, and Bash (`.sh`) when neither is clearly detected.
- Do not generate helper scripts whose only purpose is exploration, discovery, inspection, inventory, or reporting.

## Generated Structure Checks

- `.uigraph.yaml` must be valid YAML when generated.
- Generated `.json` files must parse as valid JSON.
- Generated OpenAPI, GraphQL, gRPC, SQL, Mermaid, and docs files should be structurally plausible for their declared type.
- Generated helper scripts must exist only under `.uigraph/scripts/`.
- Generated helper scripts must not overwrite unrelated files unless the user explicitly approved that behavior.

## Diagram Context Checks

- Architecture diagram `context.json` files must follow the behavior documented in `references/architecture-diagrams.md`.
- Generated diagram context should be checked against the matching skill-owned example file in `assets/templates/diagram-context/`.
- Generated `groups` should be absent unless the source evidence or user request explicitly identifies a boundary/grouping reason.
- Mermaid node IDs must match keys in `context.nodes` when node context is used, for flowchart-style diagrams.
- Sequence diagram context must key `context.nodes` on generated IDs (`participant-<id>`, `message-<row>`, `note-<row>`), never on the bare participant name.
- Sequence diagram context must not set `type` or `___internal` on `participant-*` nodes, and must not declare `groups`.
- C4 diagram context must key `context.nodes` on element aliases, must never key on a boundary id, and must not declare `groups`.
- A C4 `Rel` must have concrete elements at both ends or boundary ids at both ends, never one of each.
- Whenever a C4 diagram holds two or more boundaries at the same nesting level, every one of those boundaries must be the source or the target of at least one boundary relationship.
- C4 boundary relationships must be written in addition to the element relationships they summarise, never instead of them.
- `groups[*]` must be an object with optional `name` and `nodes`, never a bare array of node IDs.
- `groups[*].nodes` entries must reference existing `context.nodes` keys.
- `edges` keys must follow `<source>-<target>` for Mermaid edges when edge context is used.
- Context must avoid fields not supported by the converter schema, especially group `style` and node style `backgroundColor`.
- Node `data[*].type` must use one of the supported component field type names documented in `references/architecture-diagrams.md`.
- Node `shape` values must use one of the supported shape values documented in `references/architecture-diagrams.md`.
- GIF `animatedIcon` values should use one of the documented animated icon names unless a direct `src` is provided.

## Component Linking Validation

For every `components` entry under a focal point:

1. `componentId` is required and must be one of the four valid enums.
2. At least one of `componentLinkId`, `serviceName`, or `modalFields` is required.
3. If `componentId` is `component_backend-flow-diagram` and `componentLinkId` is empty: `serviceName` and `architectureDiagramName` are both required.
4. If `componentId` is `component_api-contract` and `componentLinkId` is empty: `serviceName`, `apiGroupName`, and `operationId` are all required.

## Sections That Require a Service Block

A config without a `service` block may only sync maps and frames. These sections are
rejected outright when `service.name` is empty: `apis`, `dependencies`,
`architectureDiagrams`, `testPacks`, `databases`, `queries`, `docs`, `costTags`,
`timeline`.

## Service Dependency Rules

- `dependencies` requires a `service` block; a config without a service may only sync maps and frames.
- `dependencies[*].name` is required and must be unique across all dependencies.
- `dependencies[*].service` is required and names the target service depended upon.
- `dependencies[*].service` must not equal `service.name` (no self-dependency).
- `dependencies[*].direction` is required and must be `upstream` or `downstream`. It records where the current service sits relative to the target: `downstream` means this service calls the target; `upstream` means the target calls this service. It is stored verbatim and is never inferred.
- Two dependencies may name the same target service with opposite directions, under two distinct `name` values, to declare a mutual relationship.
- `dependencies[*].criticality` is required and must be `hard` or `soft`.
- `dependencies[*].type`, when present, must be `http`, `graphql`, `grpc`, or `database`; it may be omitted.
- `dependencies[*].apiEndpointNames[*]` must each be non-empty and unique within that dependency.
- `apiGroupName`, `apiEndpointNames`, and `databaseName` describe the target service's artifacts, not the current service's; they are free-form references and are not existence-checked against local files.

## Test Case Rules

- Test cases may be inline under `testCases`, external via `testPacks[*].testCasesPath`, or both (merged).
- Do not create project test framework files for UiGraph test packs.
- Do not derive UiGraph test packs from Vitest, Jest, Pytest, or PHPUnit files unless explicitly requested.
- UiGraph test cases must use only schema fields, not framework-specific fields like `describe`, `it`, `expect`, `test`, `mock`, or `beforeEach`.
- `testCases[*].title` is required.
- `testCases[*].order` is required.
- API test cases should link to API operations with matching `apiGroupName` and `operationId` when API evidence exists, but linking is optional — omit both for a case not tied to a synced endpoint (e.g. a custom URL).
- When `type` is `api` and `operationId` is set, it must reference an existing API group and operation. A set `operationId` that does not resolve to a synced endpoint fails the sync (no silent fallback to `GET`).
- When it resolves, the test case is stored with a real link to that API spec (`apiSpecId`) plus the endpoint's HTTP method, matching UI-authored cases.
- `testCases[*].screenshots[*]` must point to an existing file that is not a directory, with a `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, or `.svg` extension. Do not invent screenshot paths — a missing file fails the whole sync.

## Cost Tag Rules

- `costTags` requires a `service` block.
- `costTags[*].key` and `costTags[*].value` are both required and must be non-empty.
- The `key`/`value` pair must be unique across the list. The same key with different values is allowed.
- `costTags` is declarative: omitting it leaves existing rules untouched, while including it makes the file own the full set — any rule not listed is deleted on sync, and `costTags: []` deletes them all.
- Do not generate `costTags` speculatively or as an empty list. Omit the section unless the user supplied real tag keys and values.

## Timeline Rules

- `timeline` requires a `service` block.
- `timeline.decisions.paths[*]` and `timeline.incidents.paths[*]` must each be non-empty strings.
- Path patterns are single-level globs. `**` is not supported and matches nothing.
- `timeline.releases.changelogPath`, when set, must point to an existing file.
- Every decision and incident file must yield a title: front-matter `title` or a `# ` heading. A file with neither fails the sync.
- A decision `status`, when present, must be `proposed`, `accepted`, `superseded`, or `deprecated`.
- Explicit dates (front matter, filename) must be `YYYY-MM-DD` or RFC 3339. A malformed date fails the sync rather than falling back.
- Changelog `##` headings must begin with a digit, optionally bracketed and optionally `v`-prefixed. Other headings, including `## [Unreleased]`, produce no events.
- Do not configure `timeline.releases.changelogPath` when the pipeline runs `uigraph release`. Both write the same `release:<version>` event and overwrite each other.

## ML Project Rules

- `ml[*].name` is required.
- `ml[*].type` must be `model` or `training`.
- A `model` project must declare `models` and must not declare `experiments`.
- A `training` project must declare `experiments` and must not declare `models`.
- `ml[*].source.type` must be `mlflow`.
- Exactly one of `ml[*].source.url` and `ml[*].source.urlEnv` must be set.
- Environment variables named by `urlEnv` and `tokenEnv` must be set and non-empty at validation time, or the sync fails before contacting anything.
- `ml[*].models[*].name` and `ml[*].experiments[*].name` are required.
- `ml` reads from a live MLflow server, not from repository files. Do not generate it without an MLflow URL from the user.

## Frame Image Rules

- If `imagePath` is provided, the file must exist.
- The CLI computes SHA256 of the image and checks with the gateway whether upload is needed.

## Doc Upload Rules

- The CLI computes SHA256 of the doc file.
- If the hash matches the gateway's stored hash, the upload is skipped.
- Supported content types for upload: `application/pdf`, `text/html`, `text/markdown`, `application/msword`, `text/plain`, `image/png`, `image/jpeg`, `image/gif`, `image/webp`, `image/svg+xml`, `video/mp4`, `video/quicktime`, `video/webm`, `audio/mpeg`, `audio/wav`, `audio/ogg`, `audio/mp4`.
