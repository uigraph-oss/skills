# Local Viewer

The `uigraph` npm package runs the viewer. Run it as `npx --yes uigraph` from the repository root, and check `npx --yes uigraph --help` instead of guessing commands or options.

- Start the viewer with `npx --yes uigraph`. It runs in the background, adds the project when it has artifacts, and opens the browser. If the viewer is already running, it adds the project to it instead. If the project has no `.uigraph.yaml`, offer to generate artifacts with `references/artifact-generation.md`.
- After generating artifacts, run `npx --yes uigraph` to add the project to the viewer.
- After editing artifacts, tell the user to refresh the browser. The viewer reads the files on every page load, so do not restart it.
- If the command reports invalid artifacts, fix them with `references/artifact-generation.md`, then run it again.
- Stop the viewer with `npx --yes uigraph stop`. Never kill its process yourself.
- Restart with `npx --yes uigraph stop`, then `npx --yes uigraph`.
- Check the viewer with `npx --yes uigraph status`. It exits with code 1 when the viewer is not running, which is not an error.
- Always give the user the URL from the command output, `http://127.0.0.1:7357/` by default, because the browser may not open, for example over SSH, in WSL, or in a container. The viewer listens only on `127.0.0.1`, so on a remote machine the user needs port forwarding, such as `ssh -L 7357:127.0.0.1:7357 <host>`.
- The viewer shows services, APIs, dependencies, architecture diagrams, database schemas, docs, and maps. It does not show test packs. The summary line the command prints counts only some of these.
- Never run `uigraph start` or `uigraph restart`, because they hold the terminal until the viewer stops.
- Never run `uigraph init` or `uigraph generate`, because they need an interactive terminal. Generate artifacts with `references/artifact-generation.md` instead.
- The command output can suggest `uigraph start`, `uigraph restart`, or `uigraph generate`. Those hints are for people in a terminal, so follow this file instead.
