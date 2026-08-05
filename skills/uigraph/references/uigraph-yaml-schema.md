# .uigraph.yaml Schema

Use this file as the source of truth before generating `.uigraph.yaml`. Do not infer the schema from examples.

## Hard Rules

- `version: 1` is required.
- `project.name` is required.
- `service.name`, `service.category`, and `service.description` are required.
- `service.repository.provider` and `service.repository.url` are required.
- `service.ownership.team` is required whenever a `service` block is present.
- `service.repository.provider` must be `github`, `gitlab`, or `bitbucket`.
- A `service` block is required to sync `dependencies`, `costTags`, and `timeline`; configs without a service may only sync maps and frames.
- `dependencies[*].name` is required, must be unique, and is the stable upsert key for the dependency edge.
- `dependencies[*].service` is required and names the target service; it must not equal `service.name`.
- `dependencies[*].direction` is required and must be `upstream` or `downstream`.
- `dependencies[*].criticality` is required and must be `hard` or `soft`.
- `dependencies[*].type`, when set, must be `http`, `graphql`, `grpc`, or `database`.
- `dependencies[*].apiEndpointNames[*]` must each be non-empty and unique within the dependency.
- `costTags[*].key` and `costTags[*].value` are both required; the key/value pair must be unique.
- `costTags` is a declarative set: omitting it leaves existing rules alone, and including it deletes any rule not listed.
- `timeline.*.paths` globs are single-level; `**` is not supported.
- `timeline.releases.changelogPath`, when set, must point to an existing file.
- `testPacks[*].testCases[*].screenshots[*]` must point to an existing image file with a `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, or `.svg` extension.
- `ml[*].type` must be `model` or `training`, and decides which of `models`/`experiments` is required and which is forbidden.
- `ml[*].source.type` must be `mlflow`, with exactly one of `url` or `urlEnv`.
- Generated API specs must be under `.uigraph/openapi/`.
- Generated architecture diagrams and context files must be under `.uigraph/diagrams/`.
- Generated database schemas must be under `.uigraph/db/`.
- Generated docs must be under `.uigraph/docs/`.
- Generated map images must be under `.uigraph/maps/`.
- Generated saved-query files (`queryFiles` and `queries[].path` SQL files) must be under `.uigraph/queries/`.
- Generated external test case files (`testPacks[].testCasesPath`) must be under `.uigraph/tests/`.
- SQL database schemas use `.sql`; NoSQL database schemas use `.json`.
- All referenced files must exist relative to `.uigraph.yaml`.
- `queryFiles` externalizes query definitions: each listed file holds a `queries:` list, merged with any inline `queries`.
- `testPacks[].testCasesPath` externalizes a pack's cases: the file holds a `testCases:` list, merged with any inline `testCases`.
- Externalized `queries[].path` (the SQL file) is still resolved relative to the working directory, not the queries file.

## Canonical Structure

```yaml
version: 1

project:
  name: my-product              # required
  environment: production       # optional

service:
  name: My Service              # required
  category: Backend             # required
  description: Short summary    # required
  repository:
    provider: github            # required: github, gitlab, bitbucket
    url: https://github.com/org/repo  # required: current git remote or user-provided URL
  ownership:                    # required when a service is present
    team: platform              # required
    email: platform@example.com # optional
  labels:                       # optional
    - backend
  integrations:                 # optional
    slack:
      url: https://example.slack.com/archives/C123456
    jira:
      url: https://example.atlassian.net/projects/ABC

apis:                           # optional
  - name: public-api            # required
    type: openapi               # required: openapi, graphql, grpc
    path: .uigraph/openapi/public-api.yaml  # required

dependencies:                   # optional; other services or datastores this service relates to
  - name: gateway-sync-api      # required: unique upsert key for the dependency edge
    service: UIGraph Gateway    # required: target service name; must not equal service.name
    direction: downstream       # required: upstream, downstream — "downstream" means this service calls the target
    criticality: hard           # required: hard, soft
    type: http                  # optional: http, graphql, grpc, database
    description: Sends synced catalog metadata to the gateway.  # optional
    apiGroupName: gateway-sync-api  # optional: API group on the target service
    apiEndpointNames:           # optional; each must be non-empty and unique
      - SyncCatalog
  - name: orders-store          # example database dependency
    service: Orders DB          # required
    direction: downstream       # required
    criticality: soft           # required
    type: database              # optional
    databaseName: orders        # optional: datastore name for database-type dependencies
  - name: storefront-consumers  # example inbound edge
    service: Storefront         # required
    direction: upstream         # the storefront calls this service
    criticality: soft           # required

architectureDiagrams:           # optional
  - name: Request Flow          # required
    path: .uigraph/diagrams/request-flow/request-flow.mmd  # required
    contextPath: .uigraph/diagrams/request-flow/context.json  # optional

databases:                      # optional
  - name: app                   # required
    dialect: postgres           # required: postgres, mysql, sqlite, dynamodb, mongodb, other
    dbType: PostgreSQL          # optional
    schemaPath: .uigraph/db/app.sql  # required

queries:                        # optional; each references a database by name
  - name: top-customers         # required: stable upsert key
    database: app               # required: must match a databases[].name
    path: .uigraph/queries/top-customers.sql  # exactly one of path or queryText
    description: Top customers   # optional
    tags: [reporting]           # optional

queryFiles:                     # optional; externalize query definitions to keep this file small
  - .uigraph/queries/reporting.yaml  # each file holds its own `queries:` list; merged with inline queries

testPacks:                      # optional; UiGraph metadata only, not Vitest/Jest/Pytest files
  - name: API Smoke Pack        # required
    type: smoke                 # required: smoke, regression, manual
    environment: staging        # optional
    releaseLabel: v1.0.0        # optional
    testCases:                  # optional; see references/test-packs-and-cases.md
      - title: Health endpoint returns 200  # required
        type: api               # required: api, manual
        order: 1                # required
        priority: p1            # optional: p0, p1, p2, p3
        apiGroupName: public-api
        operationId: healthCheck
        expectedStatusCode: 200
        screenshots:            # optional; reference images attached to the case
          - .uigraph/tests/screenshots/health-200.png  # must exist; .png .jpg .jpeg .gif .webp .svg
    testCasesPath: .uigraph/tests/api-smoke.yaml  # optional; file holds a `testCases:` list, merged with inline testCases

docs:                           # optional
  - name: Runbook               # required
    path: .uigraph/docs/runbook.md  # required
    fileType: markdown          # optional: pdf, html, markdown, doc, txt, image, video, audio, other
    description: On-call runbook

costTags:                       # optional; see references/cost-tags-and-timeline.md
  - key: team                   # required: cloud tag key, matched exactly
    value: checkout             # required: cloud tag value, matched exactly
  - key: Service                # the same key may repeat with a different value
    value: booking-api          # the key/value PAIR must be unique

timeline:                       # optional; see references/cost-tags-and-timeline.md
  decisions:
    paths:                      # single-level globs only; `**` is NOT supported
      - docs/adr/*.md
  incidents:
    paths:
      - docs/postmortems/*.md
  releases:
    changelogPath: CHANGELOG.md # must exist; omit when the pipeline runs `uigraph release`

ml:                             # optional; pulled from MLflow, not from repo files
  - name: Recommendations       # required
    type: model                 # required: model, training
    description: Ranking models for the product feed.  # optional
    ownership:                  # optional
      team: ml-platform
      email: ml@example.com
    source:
      type: mlflow              # required: must be mlflow
      urlEnv: MLFLOW_URL        # exactly one of url or urlEnv; the env var must be set and non-empty
      tokenEnv: MLFLOW_TOKEN    # optional; when set the env var must be set and non-empty
    models:                     # required for type: model; forbidden for type: training
      - name: feed-ranker       # required
        problemType: ranking    # optional: classification, regression, ranking, generation, embedding, other
  - name: Recommendations Training
    type: training
    source:
      type: mlflow
      url: http://localhost:5000  # inline alternative to urlEnv
    experiments:                # required for type: training; forbidden for type: model
      - name: feed-ranker-sweeps  # required

maps:                           # optional; see references/maps-frames-focalpoints.md
  - name: Product Map           # required
    description: Main user flows
    frames:
      - name: Home Page         # required
        imagePath: .uigraph/maps/home-page.png
        focalPoints:
          - name: Submit Button # required
            x: 120              # required
            y: 240              # required
            visibility: public  # optional: public, private
            components:
              - componentId: component_api-contract
                serviceName: My Service
                apiGroupName: public-api
                operationId: healthCheck
```

## Dependency Rules

Each dependency declares one directed relationship. `direction` records where the current service sits relative to the target and is stored verbatim — it is never inferred, normalized, or swapped. A target service that has not been onboarded is retained as an unresolved dependency and does not fail synchronization. Removing a dependency from `.uigraph.yaml` removes the stored relationship on the next sync.

## Component Link Rules

When `componentLinkId` is absent, component links must include these fields:

- `component_api-contract`: `serviceName`, `apiGroupName`, `operationId`
- `component_test-case-suite`: `serviceName`, `testPackName`
- `component_support-kb-troubleshooting`: `serviceName`, `docName`
- `component_backend-flow-diagram`: `serviceName`, `architectureDiagramName`

Names must match existing entries exactly: service name, API name, OpenAPI `operationId`, test pack name, doc name, architecture diagram name, map name, frame name, and focal point name.

## Common Invalid Output

```yaml
service:
  name: mytokens
  category: cli
  description: CLI tool
  repository:
    provider: github
    url: https://github.com/org/repo

databases:
  - name: tokens
    dialect: sqlite
    schemaPath: db/main.sql

apis:
  - name: models-dot-dev
    type: openapi
    path: openapi/models-dot-dev.yaml
```

This is invalid because:

- It is missing required `version: 1`.
- It is missing required `project.name`.
- Generated artifact paths are outside `.uigraph/`.

## Detailed References

- Service dependencies: `references/service-dependencies.md`
- Cost tags and timeline: `references/cost-tags-and-timeline.md`
- Test packs and test cases: `references/test-packs-and-cases.md`
- Maps, frames, and focal points: `references/maps-frames-focalpoints.md`
- Database schemas: `references/database-schemas.md`
- Architecture diagrams: `references/architecture-diagrams.md`
