# Lens

Run Lens only when the user explicitly asks for it. Lens reviews a GitHub pull request or the changes between two branches. When the user does not say which, ask.

- Run `npx --yes uigraph lens` with the pull request, or with `--base` and `--target` for two branches.
- Pass `--agent` with the agent you are running as: `claude`, `codex`, `opencode`, `pi`, or `cursor`. The command fails without it outside an interactive terminal. If you are none of them, ask the user which one to use.
- Start the local viewer first with `references/local-viewer.md` when it is not running. Otherwise `uigraph lens` starts it in the foreground and holds the terminal.
