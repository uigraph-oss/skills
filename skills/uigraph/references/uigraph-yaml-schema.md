# .uigraph.yaml Schema

Use this file as the source of truth before generating `.uigraph.yaml`. Do not infer the schema from examples.

## Hard Rules

- `version: 1` is required.
- `project.name` is required.
- `service.name`, `service.category`, and `service.description` are required.
- `service.repository.provider` and `service.repository.url` are required.
- `service.ownership.team` is required whenever a `service` block is present.
- `service.repository.provider` must be `github`, `gitlab`, or `bitbucket`.
- `dependencies[].name`, `dependencies[].service`, `dependencies[].type`, and `dependencies[].criticality` are required.
- Dependency names must be unique within a service.
- Dependency `type` must be `http`, `grpc`, `event`, `queue`, or `database`.
- Dependency `criticality` must be `hard` or `soft`.
- `http` and `grpc` dependencies require `api`; their optional `operations` must name operations in the provider API.
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

dependencies:                  # optional; direct upstream services only
  - name: inventory-api         # required: stable synchronization key
    service: inventory-service  # required: provider service name
    type: http                  # required: http, grpc, event, queue, database
    criticality: hard           # required: hard, soft
    description: Reserve stock before checkout
    api: inventory-api          # required for http and grpc
    operations:                 # optional; provider API operation IDs
      - getSkuAvailability
      - reserveInventory

apis:                           # optional
  - name: public-api            # required
    type: openapi               # required: openapi, graphql, grpc
    path: .uigraph/openapi/public-api.yaml  # required

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
    testCasesPath: .uigraph/tests/api-smoke.yaml  # optional; file holds a `testCases:` list, merged with inline testCases

docs:                           # optional
  - name: Runbook               # required
    path: .uigraph/docs/runbook.md  # required
    fileType: markdown          # optional: pdf, html, markdown, doc, txt, image, video, audio, other
    description: On-call runbook

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

## Component Link Rules

## Dependency Rules

Dependencies declare only direct upstream relationships. A provider that has not been onboarded is retained as an unresolved dependency and does not fail synchronization. When the provider is already onboarded, HTTP and gRPC declarations are validated against its current API specification and declared operation IDs during the consumer sync. Removing a dependency from `.uigraph.yaml` removes the stored relationship on the next sync.

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
    url: https://github.com/NazmusSayad/mytokens

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

- Test packs and test cases: `references/test-packs-and-cases.md`
- Maps, frames, and focal points: `references/maps-frames-focalpoints.md`
- Database schemas: `references/database-schemas.md`
- Architecture diagrams: `references/architecture-diagrams.md`
