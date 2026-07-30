# Service Dependencies

Service dependencies declare the relationships between the current service and other services or datastores. They are synced from the top-level `dependencies:` array in `.uigraph.yaml` to the gateway as edges owned by the current service. Every edge carries an explicit `direction` that records which way the relationship flows; it is stored verbatim and is never inferred from context. They are metadata only; nothing is fetched or introspected from the target service.

Only propose dependencies when the repository or the user describes real outbound calls, integrations, or datastore usage. Do not invent dependencies.

## Direction

`direction` describes where the **current** service sits relative to the target service.

| Value | Meaning | Read it as |
| --- | --- | --- |
| `downstream` | The current service is downstream of the target. | "I call them." |
| `upstream` | The current service is upstream of the target. | "They call me." |

Most declared dependencies are `downstream`. Use `upstream` to record an inbound consumer the current service knows about. A mutual relationship (A calls B and B calls A) is two separate entries under two distinct `name` values, one on each side or both on one side with opposite directions.

## When to Propose

- The service calls another internal service over HTTP, GraphQL, or gRPC (`direction: downstream`).
- The service reads from or writes to a database owned by another service (`direction: downstream`).
- The service integrates with a third-party service (for example a payment provider) that the user wants tracked (`direction: downstream`).
- Another service consumes this service and the user wants that inbound edge recorded here (`direction: upstream`).

## Schema

```yaml
dependencies:
  - name: gateway-sync-api      # required: unique upsert key for the edge
    service: UIGraph Gateway    # required: target service name; must not equal service.name
    direction: downstream       # required: upstream, downstream
    criticality: hard           # required: hard, soft
    type: http                  # optional: http, graphql, grpc, database
    description: Sends synced catalog metadata to the gateway.  # optional
    apiGroupName: gateway-sync-api  # optional: API group on the target service
    apiEndpointNames:           # optional; each non-empty and unique
      - SyncCatalog
  - name: orders-store          # database dependency example
    service: Orders DB          # required
    direction: downstream       # required
    criticality: soft           # required
    type: database              # optional
    databaseName: orders        # optional: datastore name for database-type dependencies
  - name: storefront-consumers  # inbound edge example
    service: Storefront         # required
    direction: upstream         # the storefront calls this service
    criticality: soft           # required
```

## Field Reference

| Field | Required | Notes |
| --- | --- | --- |
| `name` | yes | Unique across all dependencies. Stable upsert key for the edge. |
| `service` | yes | Target service name. Must not equal `service.name`. |
| `direction` | yes | `downstream` (this service calls the target) or `upstream` (the target calls this service). Stored verbatim; never inferred. |
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
- `direction` is recorded exactly as declared. Consumers of the graph render the stored value; they do not normalize, swap, or recompute it.

## Common Mistakes

- Omitting `direction`. It is required on every dependency; the sync is rejected without it.
- Using `upstream` when you meant `downstream`. `downstream` is "I call them"; `upstream` is "they call me". Almost every outbound call is `downstream`.
- Abbreviating the value as `up` or `down`. The only accepted values are the full words `upstream` and `downstream`.
- Listing the current service as its own dependency.
- Reusing the same `name` for two dependencies, including the two halves of a mutual relationship.
- Setting `type` to a value other than `http`, `graphql`, `grpc`, or `database`.
- Pointing `apiGroupName`/`databaseName` at the current service's local artifacts instead of the target service's.
- Adding dependencies without a `service` block in the config.
