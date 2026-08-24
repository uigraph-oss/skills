# C4 Diagrams

A `.mmd` file whose first diagram keyword starts with `C4`, declared in `.uigraph.yaml` under `architectureDiagrams` like any other diagram:

```yaml
architectureDiagrams:
  - name: CI/CD Platform
    path: .uigraph/diagrams/ci-cd-platform/ci-cd-platform.mmd
    contextPath: .uigraph/diagrams/ci-cd-platform/context.json
```

Write C4 when the user asks for it specifically, or when the repository already documents itself in C4 terms. `C4Container` is the level people almost always mean; `C4Context` shows whole systems with no look inside, `C4Component` shows the inside of one container.

## Supported Mermaid Syntax

The full https://mermaid.js.org/syntax/c4.html surface is supported. Write ordinary Mermaid; nothing needs a UIGraph-specific escape hatch.

```mermaid
C4Container
Person(developer, "Developer", "Pushes branches and opens pull requests")
System_Ext(chat, "Chat Platform", "Delivers deployment notifications")

System_Boundary(platform, "CI/CD Platform") {
  Container(source, "Source Control", "Git and Go", "Hosts repositories and protected branches")
  ContainerQueue(queue, "Job Queue", "Redis Streams", "Holds queued jobs partitioned by runner label")

  Container_Boundary(execution, "Execution Fleet") {
    Container(linux, "Linux Runner", "Go", "Executes jobs in ephemeral containers")
  }

  Container_Boundary(supply, "Supply Chain") {
    ContainerDb(registry, "Artifact Registry", "OCI distribution", "Stores images and build artifacts")
  }
}

Rel(developer, source, "Pushes commits to", "HTTPS")
Rel(source, queue, "Enqueues pipeline jobs on", "Redis")
Rel(linux, registry, "Pushes images to", "OCI")
Rel(registry, chat, "Announces published artifacts to", "HTTPS")

Rel(execution, supply, "Publishes signed build artifacts to", "OCI")
```

That covers every element (`Person`, `Person_Ext`, `System`, `System_Ext`, `Container`, `ContainerDb`, `ContainerQueue`, `Component`), the boundaries (`System_Boundary`, `Container_Boundary`, `Enterprise_Boundary`, braces required), and the relationships. `BiRel` and the directional `Rel_U` / `Rel_D` / `Rel_L` / `Rel_R` / `Rel_Back` all work too.

A `C4Container` diagram draws the system's own pieces as `Container`, `ContainerDb` or `ContainerQueue` with the technology filled in, never as bare `System` elements. People and third-party systems stay outside the boundary.

## Boundary Relationships

A boundary id is a valid relationship endpoint, and a `Rel` with a boundary id at **both** ends stands for that boundary as a whole. The UiGraph editor draws these when the diagram is zoomed out far enough that a boundary's contents disappear, so they are required, not decorative.

- Whenever a diagram holds two or more boundaries at the same nesting level, every one of them must be the source or target of at least one boundary relationship. A boundary alone at its level needs none.
- Write them in addition to the element relationships, never instead of them.
- The label summarises everything crossing between the two boundaries in one sentence, not a copy of one inner element's label.
- Never mix a boundary id and an element id in the same `Rel`.

## Context

Node IDs are the element aliases, so `Person(user, "User")` is keyed as `user`. Boundary ids are never context keys, and C4 diagrams never declare `groups` — nesting lives in the boundary braces. `name`, `data` and `style` behave exactly as for flowchart nodes.

The most reliable C4 pair comes from the UiGraph diagram editor: draw or adjust the diagram there and use **Export To Mermaid**, which writes the `.mmd` and its context file together.
