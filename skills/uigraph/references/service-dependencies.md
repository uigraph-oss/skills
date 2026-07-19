# Service Dependencies

Service dependencies declare the other services and datastores the current service relies on. They are synced from the top-level `dependencies:` array in `.uigraph.yaml` to the gateway as edges from the current service. They are metadata only; nothing is fetched or introspected from the target service.

Only propose dependencies when the repository or the user describes real outbound calls, integrations, or datastore usage. Do not invent dependencies.

## When to Propose

- The service calls another internal service over HTTP, GraphQL, or gRPC.
- The service reads from or writes to a database owned by another service.
- The service integrates with a third-party service (for example a payment provider) that the user wants tracked.

## Schema

```yaml
dependencies:
  - name: gateway-sync-api      # required: unique upsert key for the edge
    service: UIGraph Gateway    # required: target service name; must not equal service.name
    criticality: hard           # required: hard, soft
    type: http                  # optional: http, graphql, grpc, database
    description: Sends synced catalog metadata to the gateway.  # optional
    apiGroupName: gateway-sync-api  # optional: API group on the target service
    apiEndpointNames:           # optional; each non-empty and unique
      - SyncCatalog
  - name: orders-store          # database dependency example
    service: Orders DB          # required
    criticality: soft           # required
    type: database              # optional
    databaseName: orders        # optional: datastore name for database-type dependencies
```

## Field Reference

| Field | Required | Notes |
| --- | --- | --- |
| `name` | yes | Unique across all dependencies. Stable upsert key for the edge. |
| `service` | yes | Target service name. Must not equal `service.name`. |
| `criticality` | yes | `hard` (outage propagates) or `soft` (degraded but tolerable). |
| `type` | no | `http`, `graphql`, `grpc`, or `database`. Omit when unknown. |
| `description` | no | Short summary of what the dependency is used for. |
| `apiGroupName` | no | API group on the target service the dependency uses. |
| `apiEndpointNames` | no | Endpoints/operations used on the target service. Each non-empty and unique. |
| `databaseName` | no | Datastore name, typically paired with `type: database`. |

## Semantics

- `apiGroupName`, `apiEndpointNames`, and `databaseName` describe the target service's artifacts, not the current service's own `apis` or `databases`. They are free-form references and are not existence-checked against local files.
- A `service` block must be present for dependencies to sync. A config without a service may only sync maps and frames.
- Use `hard` when the current service cannot function correctly if the dependency is down; use `soft` for optional or gracefully degraded dependencies.

## Common Mistakes

- Listing the current service as its own dependency.
- Reusing the same `name` for two dependencies.
- Setting `type` to a value other than `http`, `graphql`, `grpc`, or `database`.
- Pointing `apiGroupName`/`databaseName` at the current service's local artifacts instead of the target service's.
- Adding dependencies without a `service` block in the config.
