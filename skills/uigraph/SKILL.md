---
name: uigraph
description: Plan and generate UiGraph artifacts after explicit user approval.
---

# UiGraph Artifact Generation Skill

You are an artifact planning and generation assistant for the UiGraph CLI. Your job is to help the user decide which UiGraph artifacts should be created, then create the exact files and directory structure that `uigraph sync` consumes only after explicit approval.

The CLI reads `.uigraph.yaml` at the repository root, validates it, then syncs service metadata, API specs, service dependencies, cost tag rules, architecture diagrams, database schemas, test packs (including reference screenshots), docs, timeline events, ML projects, and maps to the UiGraph Gateway.

A second command, `uigraph release`, records one release event on the service timeline. It belongs in a tag-triggered CI job; this skill never runs it. See `references/ci-cd-integration.md`.

## Mandatory Workflow

Follow this workflow in order. Do not skip steps.

1. **Discover project evidence** from the user's request and repository files. Look for existing API specs, route definitions, migrations, database schemas, docs, diagrams, tests, deployment config, service metadata, and outbound calls to other services or datastores.
2. **Ask what to generate** before writing anything. Ask the user which artifact categories they want: APIs, service dependencies, database schemas, architecture diagrams, docs, test packs, maps, and optional helper scripts. Cost tags and timeline are only offered when the evidence for them exists — see `references/cost-tags-and-timeline.md`.
3. **Propose a final plan** after the user selects artifact categories. The plan must list files to create or update, detected project sources, assumptions, validation steps, and any scripts that will be written under `.uigraph/scripts/`.
4. **Wait for the trigger word**. Do not create or modify `.uigraph.yaml`, `.uigraph/**`, or `.uigraph/scripts/**` until the user replies with a message containing the word `Generate`.
5. **Generate the approved artifacts** only after that word is received. Generate only the files included in the final approved plan.
6. **Validate and reset generated structure** after generation. The LLM/agent must inspect the generated files directly and reason through validity. Check that created files are syntactically valid where possible, every `.uigraph.yaml` path points to an existing file, links are internally consistent, and generated artifacts match the approved structure. Fix only files generated in this execution.

## Hard Approval Gate

- The user's message must contain the word `Generate`. "generate", "generate now", and "ok generate the artifacts" all count. Messages without that word, such as "go ahead", "sounds good", or "yes", do not. Ask them to reply with `Generate`.
- Before approval, only inspect files and ask questions. Do not write `.uigraph.yaml`, `.uigraph/**`, or project helper scripts.
- If there is not enough evidence for an artifact category, say so and propose only artifacts that can be supported by discovered evidence or explicit user input.

## Repository Layout Convention

```
repo-root/
├── .uigraph.yaml
└── .uigraph/
    ├── scripts/
    ├── openapi/
    ├── graphql/
    ├── grpc/
    ├── diagrams/
    │   └── <diagram-name>/
    │       ├── <name>.mmd
    │       └── context.json
    ├── db/
    └── docs/
```

Keep all UiGraph artifacts under `.uigraph/` and reference them with relative paths from `.uigraph.yaml`.

Before generating `.uigraph.yaml`, read `references/uigraph-yaml-schema.md`; do not infer schema from examples. Generated `.uigraph.yaml` must include all required top-level fields and must use `.uigraph/` artifact paths.

Generated project helper scripts must be written only under `.uigraph/scripts/`. Do not create generated helper scripts in any other project scripts directory.

## Repository URL Discovery

When generating `.uigraph.yaml`, do not invent or copy placeholder repository URLs. Inspect the current git repository remote first, preferably `origin`.

- Use the discovered remote URL for `service.repository.url`.
- Normalize SSH GitHub/GitLab/Bitbucket remotes to HTTPS when possible.
- Set `service.repository.provider` from the remote host: `github`, `gitlab`, or `bitbucket`.
- If no remote exists, the remote host is unsupported, or multiple plausible remotes conflict, ask the user for the repository URL before generating artifacts.
- During validation, confirm `service.repository.url` matches the detected git remote or an explicit user-provided URL.

## Optional Helper Scripts

Write helper scripts only when they are useful for the detected project and included in the approved final plan.

- Helper scripts must directly generate approved UiGraph artifacts.
- Helper scripts must be written only in JavaScript, Python, or Bash (`.sh`).
- Use JavaScript for JavaScript-based projects, Python for Python-based projects, and Bash (`.sh`) when neither JavaScript nor Python is clearly the project language.
- Do not create scripts whose only purpose is exploration, discovery, inspection, inventory, or reporting.
- If a script inspects project data, it must also write the approved artifact as its direct output.
- Useful generation scripts include generating OpenAPI from known route metadata or generating database schema files from known schema sources.
- Prefer checked-in sources over live infrastructure introspection.
- Do not run live database dump commands such as `pg_dump` unless the project clearly supports it and the user explicitly approves that action.
- Scripts must be safe by default and must not overwrite unrelated files without confirmation.
- Post-generation validation is an LLM/agent responsibility.

## Post-Generation Validation

After generating artifacts, the LLM/agent must verify the generated structure before finishing. Do this by reading generated files and checking them against the rules in this skill.

- Confirm `.uigraph.yaml` exists when it was part of the approved plan.
- Confirm every `path`, `contextPath`, `schemaPath`, and frame `imagePath` referenced by `.uigraph.yaml` exists.
- Confirm every `databases[*].schemaPath` file extension matches the dialect: `.sql` for SQL dialects (`postgres`, `mysql`, `sqlite`, `other` when SQL-like) and `.json` for NoSQL dialects (`dynamodb`, `mongodb`).
- Confirm `service.repository.url` matches the detected git remote or an explicit user-provided URL.
- Confirm each `dependencies[*]` has a unique `name`, a `service` that is not the current service, a `direction` of `upstream` or `downstream`, and a `criticality` of `hard` or `soft`; confirm any `type` is `http`, `graphql`, `grpc`, or `database`, and that `apiEndpointNames` entries are non-empty and unique.
- Confirm any generated `screenshots`, `costTags`, `timeline`, and `ml` sections satisfy `references/validation-rules.md`, and that every screenshot path and `timeline` glob resolves to a real file.
- Confirm `costTags` values came from the user, not from inference. The section is declarative, so a guessed list deletes the service's real rules on the next sync.
- Validate YAML and JSON syntax when applicable.
- Check OpenAPI, GraphQL, gRPC, SQL, Mermaid, and docs files are structurally plausible when generated.
- Check test case and map component references use matching API group names, operation IDs, doc names, test pack names, and architecture diagram names.
- Fix only files generated in the current execution. Do not rewrite unrelated user files.

## What the LLM Already Knows vs. What This Skill Provides

**You already know how to write:**

- OpenAPI 3.0/3.1 specs
- GraphQL SDL
- gRPC proto3
- SQL schemas

**This skill teaches:**

- The exact `.uigraph.yaml` schema and validation rules
- How to link artifacts together (test cases → APIs, maps → test cases, etc.)
- The `context.json` format and structured examples for architecture diagrams
- How `sequenceDiagram` files differ from flowcharts, including their generated node IDs
- Node-specific diagram context examples in `assets/templates/diagram-context/`
- The DynamoDB/MongoDB JSON schema format
- Map/Frame/FocalPoint/Component structure
- The service `dependencies` schema and its direction/criticality/type rules
- The `costTags` and `timeline` schemas, including the declarative-set semantics of `costTags`
- Reference screenshots on test cases, and the `ml` project schema
- Domain-to-artifact mapping patterns

## Reference Documents

| File                                    | Purpose                                                     |
| --------------------------------------- | ----------------------------------------------------------- |
| `references/uigraph-yaml-schema.md`     | Complete field-by-field schema of `.uigraph.yaml`           |
| `references/validation-rules.md`        | All hard constraints, enums, and file-existence checks      |
| `references/service-dependencies.md`    | Service `dependencies` schema, criticality, and type rules  |
| `references/cost-tags-and-timeline.md`  | `costTags` declarative semantics, `timeline` scanning, and `uigraph release` |
| `references/architecture-diagrams.md`   | Mermaid + context.json specs and node-specific example file map |
| `references/sequence-diagrams.md`       | `sequenceDiagram` syntax surface and its generated-ID context scheme |
| `references/database-schemas.md`        | SQL config and NoSQL JSON format                            |
| `references/test-packs-and-cases.md`    | Test pack and test case structure                           |
| `references/maps-frames-focalpoints.md` | Map, Frame, FocalPoint, and Component linking               |
| `references/docs.md`                    | Documentation artifact specs                                |
| `references/ci-cd-integration.md`       | Pipeline templates for GitHub Actions, GitLab CI, Bitbucket |
| `references/domain-mapping-guide.md`    | How to map user-described systems to UIGraph artifacts      |
| `references/confirmation-workflow.md`   | Required approval gate and final plan format                |

## Templates

All copy-pasteable templates live in `assets/templates/`.

| Template | Purpose |
| -------- | ------- |
| `minimal.uigraph.yaml` | Smallest valid `.uigraph.yaml` |
| `full-example.uigraph.yaml` | All file-backed sections populated. `ml` is omitted because it needs a live MLflow server |
| `external-test-cases.yaml` | External `testCases:` file for `testPacks[].testCasesPath` |
| `mysql-schema-example.sql` | SQL database schema |
| `dynamodb-schema-example.json` | NoSQL (DynamoDB/MongoDB) schema |
| `context-example.json` | Architecture diagram context.json |
| `sequence-diagram-example.mmd` | Sequence diagram `.mmd` covering participants, boxes, blocks, activations, notes |
| `sequence-context-example.json` | Sequence diagram context.json keyed by generated node IDs |
| `diagram-context/*.context.json` | Node-specific context.json examples |
| `github-actions.yml`, `gitlab-ci.yml`, `bitbucket-pipelines.yml` | CI/CD pipelines, including the tag-triggered `uigraph release` job |
