# Documentation Artifacts

Docs are files attached to a service.

A README, a changelog, a contributing guide, a code of conduct, a license, or any other pre-existing repository document is never a doc artifact. Never copy, derive, or reference one as a document under `.uigraph/docs/`. Write documents from what the code proves; those files remain sources you read, never output you generate.

## Configuration in .uigraph.yaml

```yaml
docs:
  - name: API Guide
    path: .uigraph/docs/api-guide.md
    fileType: markdown
    description: Public API documentation
  - name: Runbook
    path: .uigraph/docs/runbook.pdf
    fileType: pdf
    description: Operational runbook
```

## Doc Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Display name. Used for upsert matching |
| `path` | string | yes | Relative path to file |
| `fileType` | string | no | `pdf`, `html`, `markdown`, `doc`, `txt`, `image`, `video`, `audio`, `other` |
| `description` | string | no | Description |

## Content Type by Extension

When `fileType` is omitted, the CLI resolves the S3 content type from the file extension:

| Extension | Content-Type |
|-----------|-------------------|
| `.pdf` | `application/pdf` |
| `.html`, `.htm` | `text/html` |
| `.md`, `.markdown` | `text/markdown` |
| `.doc`, `.docx` | `application/msword` |
| `.txt` | `text/plain` |
| `.png` | `image/png` |
| `.jpg`, `.jpeg` | `image/jpeg` |
| `.gif` | `image/gif` |
| `.webp` | `image/webp` |
| `.svg` | `image/svg+xml` |
| `.mp4` | `video/mp4` |
| `.mov` | `video/quicktime` |
| `.webm` | `video/webm` |
| `.mp3` | `audio/mpeg` |
| `.wav` | `audio/wav` |
| `.ogg` | `audio/ogg` |
| `.m4a` | `audio/mp4` |
| anything else | `application/octet-stream` |

## Upload Behavior

1. CLI computes SHA256 of the file.
2. Calls `v1/sync/service/doc/prepare` with hash and size.
3. Gateway responds:
   - `action: skip` — file unchanged, nothing to do.
   - `action: upload` — returns presigned S3 URL and fileId.
4. CLI uploads to S3.
5. CLI calls `v1/sync/service/doc/complete` with fileId.

## Content Type Mapping for S3 Upload

The CLI resolves content type from `fileType` first, then falls back to the file extension:

| `fileType` | Content-Type |
|---------------|--------------|
| `pdf` | `application/pdf` |
| `html` | `text/html` |
| `markdown` | `text/markdown` |
| `doc` | `application/msword` |
| `txt` | `text/plain` |
| `image` | resolved from extension (default `image/png`) |
| `video` | resolved from extension (default `video/mp4`) |
| `audio` | resolved from extension (default `audio/mpeg`) |

When `fileType` is empty, the extension is used (see "Content Type by Extension" above), defaulting to `application/octet-stream`.

The content type sent to S3 must match what the gateway used when generating the presigned URL, or S3 returns `403 SignatureDoesNotMatch`.
