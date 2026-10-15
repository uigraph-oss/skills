# Confirmation Workflow

Full generation plans first, asks the user one round of questions, in as many tool calls as the question tool needs, and writes only after the user answers. Before the user answers, only read files, ask no questions before the confirmation questions, and do not write `.uigraph.yaml` or anything under `.uigraph/`.

## Required Order

1. Explore the repository and plan the candidates.
2. Ask the user what to generate, in one round of questions.
3. Generate what the user approved. The answer is the approval: do not propose a final plan or wait for a `Generate` word. Ask again only as described in [Ask the User](#ask-the-user).

## Files You Never Read

Never read generated directories or secret files, and never cite them as evidence.

- Directories: `.git`, `node_modules`, `dist`, `build`, `out`, `.next`, `.nuxt`, `.turbo`, `.cache`, `coverage`, `target`, `vendor`, `.venv`, `venv`, `__pycache__`, `.terraform`, `.claude`.
- Files: `.env` and `.env.*` except `.env.example`, `.env.sample`, and `.env.template`; keys and certificates (`.pem`, `.key`, `.p12`, `.pfx`, `.jks`, `.keystore`, `.crt`, `.cer`, `.der`); SSH keys such as `id_rsa` and `id_ed25519`; `credentials*`; `secret.*` and `secrets.*`; `.npmrc`, `.netrc`, `.pypirc`; `*.tfstate` and `*.tfstate.backup`.

## Explore

Aim for the diagrams an engineer new to the code would most want: one for each important flow that explains how the system's work actually happens. Every candidate must be something the code proves exists. Spend most of the exploration finding and tracing the important flows, not reading files one by one.

Read `references/architecture-diagrams.md` first. Read `references/sequence-diagrams.md` or `references/c4-diagrams.md` only if you propose that type.

1. **Read existing artifacts.** When the repository has `.uigraph.yaml` or `.uigraph/`, read `.uigraph.yaml`, `.uigraph/services/*.yaml`, and the files under `.uigraph/` first.
2. **Map the repository.** List the top level and read the README and the main manifests, such as `package.json`, `go.mod`, `pyproject.toml`, and `docker-compose.yml`. Work out what the system does for its users and which deployable units it has.
3. **Find the entry points.** Search instead of reading whole folders: route and handler registrations, queue publishers and consumers, scheduled jobs, webhook handlers, CLI commands, and the main entry point of each deployable unit.
4. **Rank the flows.** Put first the flows that matter most:
   - The core job: the flow the product exists to run, such as placing an order, syncing costs, or generating a report.
   - Risk: flows that move money, grant access, or write important data.
   - Reach: flows that cross a boundary, such as another service, a queue, a third-party API, or several datastores.
   - Complexity: flows with many steps, retries, fan-out, or state changes that are hard to follow in the code.
5. **Trace the top flows.** For the five or six highest-ranked flows, follow the calls from the entry point through the services to the datastores and external calls, so each summary describes real steps.
6. **Note the evidence for the other categories:** existing API specs, migrations and database schemas, tests, deployment config, service metadata, CODEOWNERS, and calls to other services or datastores. `references/domain-mapping-guide.md` shows which artifacts common domains call for.

## Diagram Candidates

| Evidence in the repository | Diagram | Mermaid type |
| --- | --- | --- |
| The service, its datastores, and every outbound dependency found in code or config | System context. Always propose it and recommend it. | `flowchart` |
| An important API flow such as authentication, checkout, or ingestion. One per flow, not one per endpoint. | Request lifecycle | `sequenceDiagram` |
| A worker, queue consumer, scheduled job, or webhook handler | Background flow | `flowchart` or `sequenceDiagram` |
| An entity with a status enum or a transition table | State machine | `stateDiagram-v2` |
| Major packages or modules and how they call each other, when that is not obvious from the folder names | Internal components | `flowchart` |
| Kubernetes manifests, Terraform, or similar config with several deployable units or real infrastructure | Deployment | `flowchart` |

Use `C4Container` or `C4Component` only when the repository already documents itself in C4 terms.

- Propose at most 15 candidates. Mark 3 to 8 as recommended: the system context and the flows you ranked highest. A repository with one small script deserves one or two candidates.
- Prefer a diagram of a specific important flow over a generic overview. Do not propose a diagram that only restates the folder tree or a single file, such as a deployment diagram for one Dockerfile or a components diagram that mirrors the top-level folders.
- Every candidate has:
  - a unique kebab-case `id` of at most 48 characters, which becomes the diagram directory name
  - a plain `name` of at most 60 characters
  - a Mermaid `type`: `flowchart`, `sequenceDiagram`, `stateDiagram-v2`, `C4Container`, or `C4Component`
  - a one-line `summary` of at most 160 characters saying what the diagram shows
  - 1 to 4 `evidence` files that exist and that you actually read
- Name each diagram for what it shows, such as "Cost sync from AWS" or "GitHub webhook handling". Never use generic names such as "Diagram 2".
- Never invent a flow, service, or dependency. When evidence is thin, propose fewer diagrams.
- When a document (README, comments, docs) claims something, verify it against the code before you cite it.
- Do not propose a diagram that already exists: a directory under `.uigraph/diagrams/` or an entry under `architectureDiagrams` in `.uigraph.yaml` or `.uigraph/services/*.yaml`.

## Existing Document Candidates

The user can add the repository's own Markdown and HTML documents to the viewer as they are. Find them with a pattern such as `**/*.{md,markdown,html,htm}`.

- Propose at most 10 documents that help an engineer understand or operate the service: the README, architecture notes, guides, runbooks, API docs, and decision records.
- Leave out changelogs, licenses, codes of conduct, contributing guides, issue and pull request templates, generated files, test fixtures, and documents of vendored code.
- Do not propose a document that `.uigraph.yaml` already lists under `docs`, and never propose a file under `.uigraph/`.
- Give each one its repository-relative `path` ending in `.md`, `.markdown`, `.html`, or `.htm`, a plain `name` of at most 60 characters, and a one-line `summary` of at most 160 characters. Propose none when no document is worth adding.

## Other Categories

- **Default set:** the `service` block, APIs, dependencies, docs, databases, and queries. Each is generated when the repository has evidence for it.
- **Offered:** test packs, maps, and helper scripts. Each is generated only when the user chooses it.
- **Offered only with evidence:** cost tags and timeline. See `references/cost-tags-and-timeline.md`.

## Repository URL and Owning Team

`.uigraph.yaml` needs `service.repository` and `service.ownership.team`. Find them with the Repository URL Discovery and Owning Team rules in `references/artifact-generation.md`. When either cannot be found, ask for it with the confirmation questions. Never invent one.

## Ask the User

First write the candidates in your message:

- the deployable services, when there are several, with the main one, which goes in `.uigraph.yaml`
- the default categories that have evidence and will be generated

Then ask these questions through the question tool, as `SKILL.md` requires:

1. **Diagrams.** The user ticks the diagrams to create.
2. **Existing documents.** The user ticks the documents to add. Skip this question when there are no document candidates.
3. **Services,** when there are several: whether the split and the main service are right.
4. **Other categories.** Offer test packs, maps, and helper scripts, and offer cost tags and timeline only when their evidence exists, as `references/cost-tags-and-timeline.md` defines it.
5. **Repository URL, owning team, and database type,** only the ones you could not find or decide. See `references/database-schemas.md` for the database type.
6. **Focus.** Ask whether there is anything in particular to focus on. The answer is optional.

Fit the questions to the tool's limits, using as few calls as they allow:

- Ask diagrams and documents as multi-select questions with one checkbox for each candidate. Put the name in the label and what it shows in the description, and add `(Recommended)` to the label of each recommended diagram. Never ask by number, by range, or by a list in your message. Use as many questions and calls as the candidates need.
- For the other questions, make the default the first option and mark it recommended: the proposed services, the default set with no extra categories, and no focus.
- Take the repository URL, the team, and the focus as free-text answers. Offer as options only values the repository shows, such as a ranked flow for the focus; never invent one. When there is no real value to offer, give honest options such as "I'll type it" and re-ask if no text comes back.
- Keep headers short, such as Diagrams, Docs, Services, Extras, Repo URL, Team, and Focus.

Generate only the diagrams and documents the user ticked. Choosing the defaults approves the default set. When an answer is unclear, or leaves a repository URL or team you asked for unanswered, ask again for just that through the tool before writing anything.

## Generate

Generate with `references/artifact-generation.md`, using exactly the approved diagrams, documents, and categories:

- Generate each approved diagram at `.uigraph/diagrams/<id>/<id>.mmd` with a `context.json`, using the listed Mermaid type. Generate no other new diagram except the focus diagram below. When the user approved none and named no focus, generate no new diagram.
- Add each approved document under `docs` with the listed name and its current path, and set `fileType` to `markdown` or `html`. Do not copy, move, or rewrite these files.
- Keep the diagrams that already exist, and update them when the code changed.
- When the user named a focus and it names a flow the code shows, add one diagram for it. If the code does not show it, leave it out and say so in your summary. Never invent a flow.
- When artifacts already exist, keep what is still correct and update the rest to match the current code.
