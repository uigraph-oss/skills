# Local Viewer

The `uigraph` npm package runs the viewer. Run it as `npx --yes uigraph` from the repository root, and check `npx --yes uigraph --help` instead of guessing commands or options.

- Start the viewer with `npx --yes uigraph`. It runs in the background, adds the project when it has artifacts, and opens the browser. If the project has no `.uigraph.yaml`, offer to generate artifacts with `references/artifact-generation.md`.
- Restart with `npx --yes uigraph stop`, then `npx --yes uigraph`.
- Never run `uigraph start` or `uigraph restart`, because they hold the terminal until the viewer stops.
- Never run `uigraph init` or `uigraph generate`, because they need an interactive terminal. Generate artifacts with `references/artifact-generation.md` instead.
