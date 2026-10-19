# C15 pipeline spike: results

A throwaway prototype built by a subagent during C15 planning (2026-10-19, at the user's request, Q6). The code lives in Claude's scratchpad only and is never merged. Environment:

- Vite **8.3.4** (Rolldown);
- Playwright 1.64.1 with the cached Chromium 1248, run on Node 24.15.0, Chromium only;
- no sudo, no system or global installs, and the npm cache kept inside the spike directory.

Fixture:

- a fake showcased repo, used as the Vite `root`. It has its own `index.html`, a `vite.config.ts` that throws if loaded, `public/`, and a `postcss.config.js` that leaves a marker;
- theme stylesheets in a "CLI cache" directory outside the root, whose `url()`s point at assets in a "theme package" directory, also outside the root;
- a module app entry outside the root.

## Verdict per question

1. **The programmatic build works.** The config was `configFile: false`, `root` at the repo, `base: './'`, `publicDir: false`, `envDir: false`, `cacheDir` outside the repo, and `build.write: false`. The input was a virtual `index.html` whose id is `<root>/index.html`, supplied by our plugin's `resolveId`/`load`.
   - The repo's `index.html`, `vite.config.ts` and `public/` were never read (they were made unreadable, `chmod 000`).
   - The inline classic script is kept as is.
   - **The result is one object `{ output }`**, not an array:
     - chunks carry `fileName` and `code: string`;
     - assets carry `fileName` and `source: string | Uint8Array`;
     - the mapping is `{ path: fileName, contents: code | source }`.
2. **`?inline` CSS from outside the root works in builds from a sub-path.**
   - `url()`s become `new URL('<asset>', import.meta.url).href`, resolved against the chunk. Assets under 4 KB are inlined as `data:`.
   - A constructed sheet in the shadow root and a document-level fonts sheet (`document.adoptedStyleSheets`) both loaded their font and images at `/sub/path/`. No fix is needed.
3. **Showcases each get their own chunk**, and their CSS becomes `<link>`ed assets. `__vite__mapDeps` names are `./`-relative. Minified output (the default) has no paths at all.
   - **Leak:** with `minify: false`, Rolldown emits `//#region <path relative to process.cwd()>`. With cwd `/`, that is a machine path *without* a leading slash, which a no-absolute-paths check misses. `output.comments: false` doesn't remove these comments.
   - **Fix:** set `rollupOptions.cwd = repoRoot`. The comments then read `src/…` and `../…`.
4. **`NODE_ENV` matters beyond `mode`.** With `NODE_ENV=development`, a `mode: 'production'` build minifies, but `isProduction`, `import.meta.env.PROD` and the `process.env.NODE_ENV` replacement are all development values. A `define` alone fixes only the replacement.
   - **Fix for builds:** set `process.env.NODE_ENV = 'production'` for the whole `build()` call, and restore it in `finally`. When the variable was unset before, restoring means **deleting** it, because Vite leaves it set.
   - **Dev has the mirror problem** under `NODE_ENV=production`. That case is fixed by setting `development` around `createServer`/`listen` together with an explicit `define`. `NODE_ENV=test` (as set by Vitest) is harmless.
5. **The repo's PostCSS config processes every CSS file that goes through Vite's CSS pipeline:** the theme sheets and the module's CSS (`?inline`) as well as the showcase CSS.
   - An inline `css.postcss` would turn it off for showcases too.
   - **Fix (verified): our own CSS never enters `vite:css`.** A virtual module is generated as JS, and each `url()` in the CSS becomes an asset `import` (handled by `vite:asset`) concatenated into the string.
   - Result: the same asset handling (`new URL(…, import.meta.url)`, small assets inlined), no PostCSS marker, the sub-path test passes, no paths leak, and no `fs.allow` entries are needed in `dev`.
6. **Dev server:**
   - the dev base is `/`;
   - the default `appType: 'spa'` serves the repo's own `index.html` for any unknown path. Fix: `appType: 'custom'` plus our own middleware for `/` and `/index.html` (`server.transformIndexHtml`), registered in `configureServer`; unknown paths then give 404;
   - module files and `?inline` CSS outside the root load without `fs.allow` (they are in the module graph);
   - **`url()` targets of `?inline` CSS outside the root get 403** unless their directories are added to `fs.allow` (together with the repo root). With our own pipeline (item 5) they load without any entry.
   - **Entry `src`:** neither `virtual:…` (works in build only) nor `/@id/__x00__…` (works in dev only) works in both. A fake root-absolute path (`/@tsshowcase/entry.js`) that our plugin resolves to the virtual id works in both.
7. **The live channel works.**
   - `server.ws.send('tsshowcase:…', payload)` reaches `import.meta.hot.on`.
   - `server.environments.client.reloadModule(node)` sends an HMR update the client can accept. If it isn't accepted, the page fully reloads with fresh content.
   - `invalidateModule` without an update gives fresh content on the next page load.
   - For theme updates with our own pipeline, the update path is: the CLI rewrites the file → `reloadModule` on the themes virtual module → the client accepts → `replaceSync`. The assets still resolve afterwards.
   - `server.reloadModule(…)` (the deprecated mixed-graph API) crashes; use the environment API.
   - `preprocessCSS` does **not** rewrite `url()`s.
8. **`tss-assets/` in dev works:** a middleware registered in `configureServer` runs before Vite's own. Traversal safety came from the handler (`sirv` gives it).
9. **Other surprises:** `postcss-load-config` sets `NODE_ENV=development` in the process when it is unset. In builds our guard has already set it, so this doesn't apply there. Rolldown's lazy-init wrappers are harmless.

**Not verified:** Firefox and WebKit, a dev server under a non-root base, `file://`, Windows paths, real React with `plugin-react`, and a `url()` rewrite built on `postcss-value-parser` (the spike used a regex).

## Plan changes (applied to the C15 draft, round 2; Q1 approved as D144)

- **Our own CSS pipeline (Q1):** theme and module stylesheets bypass Vite's CSS pipeline and the repo's PostCSS. Their `url()`s become asset imports.
  - `fs.allow` needs no additions, and the theme asset collection is dropped.
  - The dev theme update becomes an HMR reload of the themes module, so the `tsshowcase:theme-stylesheets` live event is no longer needed.
- **Entry `src`:** `/@tsshowcase/entry.js`. Set `appType: 'custom'` and serve the HTML from our own middleware.
- **Build output:** a single `{ output }` object. `rollupOptions.cwd = repoRoot` is enforced.
- **`NODE_ENV`:** guarded around `build()`, with a restore that deletes the variable when it was unset. Dev: Q8.
- **HMR:** use `server.environments.client.reloadModule`/`invalidateModule` only.
