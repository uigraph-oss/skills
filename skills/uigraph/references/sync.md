# Sync

## Target

Choose the target in this order:

1. If the user names a target, use it, even if an earlier sync in the conversation used another one.
2. If the conversation makes the target clear, use it. Starting the local viewer in this conversation means the local viewer. An earlier sync to UIGraph Cloud or a self-hosted gateway means that same target.
3. Otherwise, ask the user whether to sync to the local viewer or to UIGraph Cloud or a self-hosted gateway.

Sync to one target at a time, and sync to another only when the user asks.

## Commands

- **Local viewer:** `npx --yes uigraph sync`.
- **UIGraph Cloud or a self-hosted gateway:** `uigraph-cli sync`, with `--enterprise` for UIGraph Cloud or `--api-url` for a self-hosted gateway. Read `references/ci-cd-integration.md` for installation and `UIGRAPH_TOKEN`.
