# Sequence Diagrams

A `.mmd` file starting with `sequenceDiagram`, declared in `.uigraph.yaml` under `architectureDiagrams` like any other diagram:

```yaml
architectureDiagrams:
  - name: Checkout Sequence
    path: .uigraph/diagrams/checkout-sequence/checkout-sequence.mmd
    contextPath: .uigraph/diagrams/checkout-sequence/context.json
```

## Supported Mermaid Syntax

The full https://mermaid.js.org/syntax/sequenceDiagram.html surface is supported. Write ordinary Mermaid; nothing needs a UIGraph-specific escape hatch.

```mermaid
sequenceDiagram
  autonumber
  title: Checkout
  box transparent Frontend
    participant U as User
    actor Ops
  end
  participant DB@{ "type": "database", "alias": "Orders DB" }

  U->>+DB: place order
  Note right of U: idempotency key
  alt in stock
    DB-->>-U: confirmed
  else sold out
    DB--xU: rejected
  end
  loop nightly
    Ops->>DB: reconcile
  end
```

That covers participants and actors, `as` and `@{ }` aliases, every arrow token, activation (`+`/`-`), blocks (`alt`/`else`, `loop`, `opt`, `par`, `critical`, `break`, `rect`), `box` bands, notes, `autonumber`, and `title`. `create` / `destroy participant`, `activate` / `deactivate`, `link`, and `accTitle` / `accDescr` all work too.

## Context

Add `context.json` when there is real per-participant metadata to attach. Key it by participant:

```json
{
  "nodes": {
    "participant-DB": {
      "name": "Orders Database",
      "style": { "color": "#2563eb" },
      "data": {
        "Owner": { "type": "Text Input", "value": "Data Team" }
      }
    }
  }
}
```

- The key is `participant-<identifier>`, using the identifier rather than the display alias: `participant U as User` and `participant DB@{ "alias": "Orders DB" }` give `participant-U` and `participant-DB`.
- `name` and `data` behave exactly as for flowchart nodes.
- `style.color` sets the participant's indicator and lifeline color. Omit it to use the default `#E2E8F0`.
