# Cost Tags and Timeline

Two optional `.uigraph.yaml` sections that describe a service beyond its code artifacts:
`costTags` attributes cloud spend to the service, and `timeline` builds its dated history
from files already in the repository. Both require a `service` block.

Neither section should be invented. Propose `costTags` only when the user states the tag
keys their cloud resources carry, and `timeline` only when the matching files already
exist in the repository.

## Cost Tags

Each entry is a match rule. A cloud resource is attributed to the service when one of its
tags matches a rule's `key` and `value` exactly — matching is case-sensitive on both.
Multiple rules are OR-ed.

```yaml
costTags:
  - key: team
    value: checkout
  - key: Service
    value: booking-api
```

| Field | Required | Notes |
| --- | --- | --- |
| `key` | yes | Cloud tag key, matched exactly. Non-empty. |
| `value` | yes | Cloud tag value, matched exactly. Non-empty. |

### The section is declarative — warn the user

This is the behavior most likely to surprise someone:

| Config | Effect on sync |
| --- | --- |
| `costTags` omitted | Not managed by the CLI. Existing rules are left alone. |
| `costTags` with entries | The file owns the full set. Any rule in UIGraph that is not listed **is deleted**. |
| `costTags: []` | **Every rule on the service is deleted.** |

Never generate `costTags` speculatively or as an empty list "for later". Adding the
section to a service whose tag rules were created in the UI silently deletes them on the
next sync. If the user cannot name the tag keys and values, omit the section.

### Common mistakes

- Guessing tag keys from the repository instead of asking. Tag keys come from the cloud
  account, not from the code.
- Casing drift: `Service` and `service` are different keys, `production` and
  `Production` are different values.
- Duplicating the same key/value pair. The pair must be unique; the same key with
  different values is fine and is the normal way to cover a service spanning several
  tag values.
- Emitting `costTags: []` to mean "none". Omit the section instead.

## Timeline

Scanned on every `uigraph-cli sync`. Each source file becomes one event, upserted by a
`sourceRef` derived from the file path or the version — never from the title — so
retitling a file updates the event instead of duplicating it.

```yaml
timeline:
  decisions:
    paths:
      - docs/adr/*.md
      - docs/decisions/*.md
  incidents:
    paths:
      - docs/postmortems/*.md
  releases:
    changelogPath: CHANGELOG.md
```

All three sub-sections are optional. Declare only the ones whose files exist.

### Path patterns

- Globs are **single-level**. `docs/adr/*.md` works; `docs/**/*.md` matches nothing.
  Recursive patterns are not supported — list each directory as its own pattern.
- Patterns resolve relative to the working directory the CLI runs in, the same as every
  other path in the config.
- Directories matched by a pattern are skipped. Overlapping patterns are de-duplicated.
- A pattern that matches nothing is not an error; it just contributes zero events. Before
  proposing a pattern, confirm it actually matches files in the repository.

### What is read from a decision file

| Event field | Source, in order |
| --- | --- |
| Title | front-matter `title` → first `# ` heading. **Required**; a file with neither fails the sync. |
| ADR number | front-matter `adr` → leading digits of the filename |
| Status | front-matter `status` → first non-empty line under a `Status` heading |
| Summary | front-matter `summary` → first paragraph under `Context` → first paragraph under `Decision` |

`status`, when present, must be one of `proposed`, `accepted`, `superseded`,
`deprecated` (compared lower-cased). Anything else fails the sync.

### What is read from an incident file

Title follows the same rule. Summary is front-matter `summary` → first paragraph under a
`Summary` heading → first paragraph under an `Impact` heading.

Heading lookups match the heading text at any level, case-insensitively.

### Dates

Resolved in order: front-matter `date` → a `YYYY-MM-DD` in the filename (**incidents
only**) → the commit that added the file → the file's mtime. Accepted formats are
`YYYY-MM-DD` and RFC 3339; a malformed explicit date is a hard error, not a
fall-through.

### Changelog releases

`timeline.releases.changelogPath` must point to an existing Keep-a-Changelog file. Only
`##` headings that begin with a digit are recognised:

```markdown
## [1.4.0] - 2026-07-14
## 1.4.0 - 2026-07-14
## [v1.4.0]
```

Brackets, the leading `v`, and the date are each optional; `-`, `–`, and `—` all work as
the separator. A section with no date falls back to the changelog file's mtime.
`## [Unreleased]` and headings like `## Release 1.4.0` are skipped.

If the repository's changelog does not use this heading format, say so rather than
configuring `changelogPath` and letting it silently produce nothing.

### Source links

Events carry a link back to their file, built from `service.repository.url` and the
current branch. Only `github` and `gitlab` providers have a known layout; a `bitbucket`
repository syncs the events without a link.

## `uigraph-cli release` and the changelog compete

`uigraph-cli release` is a separate CLI command run from a tag-triggered CI job. It records
one release event and exits. It keys the event the same way the changelog scanner does —
`release:<version without a leading v>` — so the two sources write the **same event** and
whichever runs last overwrites the other.

- The pipeline cuts releases through CI → recommend `uigraph-cli release` and **omit**
  `timeline.releases` from the config.
- No release pipeline → keep `timeline.releases.changelogPath`.

Never configure both. The skill does not generate release notes or run the command; see
`references/ci-cd-integration.md` for the pipeline job.

## When to propose these sections

- `costTags`: the user names the cloud tag keys and values their resources carry, and
  wants spend attributed to this service.
- `timeline.decisions`: the repository already has ADRs or decision records.
- `timeline.incidents`: the repository already has postmortems.
- `timeline.releases`: the repository has a changelog in the recognised format **and**
  no CI job runs `uigraph-cli release`.
