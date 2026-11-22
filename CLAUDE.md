# esimv1-viewer

Project-specific orientation only. House policy (devlog, git, UX, writing, deploy) comes from the
harness and is not repeated here.

## What this is

Svelte 5 components that draw an esim run folder. No server, no router, no accounts: those are the site, `anselm/esimv1`.

## Where things are

- `src/*.svelte` and `src/*.js`; `src/index.js` exports them; `src/config.js` holds the three addresses the components
  need outside a run folder.
- On canada1 the working clone is `~/projects/2026/earth-simulator-1/viewer`; the hub's site reads it from source
  through a Vite alias, so a change here shows on the hub's dev server at once. There is no build of its own yet.
- The run folder's layout is documented in `anselm/esimv1-core`, `docs/system/projects-and-runs.md`.

