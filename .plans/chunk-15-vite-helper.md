# Chunk plan C15: Vite pipeline helper (`@ofzza/tsshowcase-base-vite`)

> Status: **APPROVED** (round 4, 2026-10-19), together with C2 amendment B8, C3 amendment C3-A2, the C6 amendment and super-plan R9, all applied.
>
> - All open questions are resolved (Q1–Q8: D138–D144).
> - The pipeline spike (`15a-c15-spike-results.md`) is applied.
> - The first fresh-eyes review (`15b-c15-plan-review.md`, S1–S16) is applied, with the user's decisions (S15: D145; S3 rewords D143).
> - The second review (`15c-c15-plan-review.md`, T1–T18) is applied, with the user's decisions (T15: D146; T16: D147; T17: D148; T18: keep the `<style>` fallback).
> - Amendments to C2 (B8), C3 (C3-A2) and C6, plus super-plan R9, are listed at the end and applied to those documents (approved with this plan).
>
> - **Redesign amendment** (the representation redesign, `08b-representation-redesign.md`, D160–D210; plan 9 of its application; review `15d-c15-redesign-review.md`, questions QA–QE and Fix 1–16): **approved 2026-10-21**. `prepareData(Representation)` keeping `revision` and `tests`; `selectDocs` and rendered docs as lazy chunks in builds and rendered on request in `dev` (D210); showcase loaders per file, `loadShowcase(file, exportName)` (D195); `applyTestUpdate` over `Representation.tests`; one shared revision and one hello (D186); the dogfood layout and the runtime test project (C1 A4); details in "Redesign amendment" below.
>
> Decisions referenced as `D<n>` are in AGENTS.md §3.

## Goal

The reusable Vite build and dev pipeline that modules call from their `build(ctx)` and `dev(ctx)` (D43), plus a small framework-agnostic **browser runtime** that their apps call (H28).

A module brings its framework plugins and its app entry. `-base-vite` does everything that is the same for every framework:

- assembling and protecting the Vite config;
- the virtual `index.html`;
- data embedding, the rendered docs and the showcase loaders;
- its own CSS pipeline for theme and module stylesheets;
- theme state and the document-level page styles;
- the dev server and the live channel;
- turning the build output into `BuildFile[]` (D64).

It knows nothing about React (C10 wraps it) and nothing about the CLI's internals: it sees only `BuildContext` and `DevContext`.

## Requirements this plan answers

The super-plan's C15 section and the AGENTS "Open points" row for C15:

- the runtime API (H28);
- the showcased repo's own bundler config (K18, D138);
- the build output as `BuildFile[]` (`write: false`, Rolldown);
- `fs.allow`;
- dev stylesheet updates (`onThemeStylesheetsUpdate`);
- stylesheet asset processing (our own pipeline, D144);
- the `index.html` meta tags and the inline selection script;
- `THEME_STORAGE_KEY` being shared per origin (L19);
- embedding only the data the app reads (L20);
- serving `docAssetsDir` (D125);
- *redesign:* the `Representation` as input, rendered docs (only rendered docs reach the output, D187, E2), showcases per (file, export) (D195), one shared revision (D186);
- an explicit `mode`/`NODE_ENV` (D94);
- a dev server that doesn't rely on `process.env.NODE_ENV` (C7 §7, D143).

## 1. Where the code lives

```
packages/tsshowcase-base-vite/src/
  index.ts                  Node entry: re-exports buildSite, startDevServer and the ViteApp types by name (C1 §8)
  pipeline/                 exported area (README): buildSite.ts (§7), startDevServer.ts (§8), each with its test;
                            ViteApp.ts (type-only: ViteApp; docs only, D162) (§2)
  config/                   internal: assembling, merging and protecting the Vite config (§3)
  virtual/                  internal: the virtual-modules plugin (§4)
  css/                      internal: our own CSS pipeline (§4.1)
  html/                     internal: index.html and the inline selection script (§5)
  docs/                     internal: rendering the selected docs (§4.2)
  output/                   internal: Rolldown output → BuildFile[] (§7)
  env-guard/                internal: NODE_ENV handling for builds (§7)
  dev-server/               internal: middlewares (HTML, docs, tss-assets/), live channel, snapshot store (§8)
  runtime/                  browser entry (subpath `./runtime`), its own tsconfig with the `dom` lib (§6)
    index.ts                re-exports the runtime's public symbols by name
    data/  live/  docs/  showcases/  stylesheets/  theme/  document/    exported areas (README each), one file per concept (§6);
                            incl. the type-only data/PreparedData.ts and docs/EmbeddedDoc.ts (§2), and data/applyTestUpdate.ts
    internal/               virtual.ts (the only importer of virtual ids and user of import.meta.hot) and selectTheme.ts (§5); no public symbols
  client/                   client.d.ts: ambient declarations for the virtual modules and `?tss-css` imports (subpath `./client`, types only), README
```

- **Everything lives under `src/`** (S13), the package's dogfood scope (C1 §8, `scope: ['src']`). The runtime sits in its own directory with its own `dom` tsconfig.
- **Dogfood layout (C1 §8, A4; D161–D163, D200):** public concepts live in exporting files named after their main export, in area directories with a README that documents them; files declaring functions get a test named after them (`buildSite.test.ts`, `loadDoc.test.ts`); type-only files need only the README. Nothing here is a component or a hook, so no showcases are required (D162). Entry files (`src/index.ts`, `src/runtime/index.ts`) only re-export, by name. Internal directories declare no public symbols and need nothing from `check`, but have tests (C1 §4, §10; D56's trigger table). `client.d.ts` lives in its own area (`src/client/`, with a README), not directly in `src/`, so it fits the layout whether or not C8 counts ambient `declare module` declarations as symbols (an open C8 point).
- **Exports:**
  - `.` — Node only: the pipeline;
  - `./runtime` — browser only: imported by the module's app;
  - `./runtime` also re-exports the `PreparedData` and `EmbeddedDoc` types; `.` re-exports them too, next to `ViteApp`;
  - `./client` — types for `virtual:tsshowcase/*` and `*.css?tss-css`, referenced from a module's app tsconfig. `tsc` never emits a hand-written `.d.ts`, so the build copies `src/client/client.d.ts` to `dist/client/client.d.ts` with the sidecar copy step (C1 A1, `15d` Fix 5); the subpath's conditions are `source` → `src/client/client.d.ts` and `types` → `dist/client/client.d.ts` (C1 §8: every subpath carries `source`).
- **The runtime only works inside this pipeline.** It imports the virtual modules, so it runs only inside a `-base-vite` build or dev server. That is intentional: it is the runtime *of this pipeline*.
- **Runtime dependencies:**
  - `vite@^8`, a real dependency: it ships with the CLI (D4's "bundler plugins ship with the CLI/module"), and the showcased repo is never tied to Vite. Vite 8 needs Node `^20.19 || >=22.12`, which our floor (D49) satisfies;
  - `@ofzza/tsshowcase-api`;
  - `postcss` and `postcss-value-parser` (§4.1);
  - `sirv` (§8).
- **The runtime is consumed by Vite, not by Node.** `./runtime` is published as built ESM (`dist/runtime/*.js`), and Vite bundles it into the app like any other dependency. It is excluded from `optimizeDeps`, so that its virtual-module imports resolve through our plugin.
- **Dogfooding (D41):** the public symbols are `buildSite`, `startDevServer`, the `ViteApp` types and the runtime's symbols (§6); each is documented in its area README, the functions are tested by name (above).
- **The virtual-module adapter (`15d` Fix 4):** `runtime/internal/virtual.ts` is the only file that imports the `virtual:tsshowcase/*` ids, and the only one that calls `import.meta.hot` (`accept` only works in the module that imports the accepted dependency). It re-exports the values and offers callback registration (`onThemesUpdate`, `onShowcasesUpdate`, `onMessage`, `send`); the concept files use it. The runtime tsconfig includes the `vite/client` types.
- **Tests of runtime code (C1 A4, `01e`; K7):** runtime tests `vi.mock` the adapter (`runtime/internal/virtual.ts`), which also covers `import.meta.hot` (undefined under jsdom); the runtime project additionally aliases the `virtual:tsshowcase/*` ids to stub modules in `packages/tsshowcase-base-vite/test/virtual/` (outside `src/`, so outside the dogfood scope) so that the adapter itself resolves in its own test. No `exempt` is needed.
- **Two TypeScript projects (`15d` Fix 3):** a runtime build project (`src/runtime/`, `dom` lib, emitting `dist/runtime/`) and a Node build project (the rest of `src/`, emitting `dist/`) that **references** it (composite, C1 §3). Node code imports the runtime's import-free files through the reference: `dev-server/` uses `runtime/data/applyTestUpdate.ts`, `html/` uses `runtime/internal/selectTheme.ts`, and `pipeline/ViteApp.ts` imports the `PreparedData` and `EmbeddedDoc` types; each file is compiled once, by the runtime project. "Import-free" means no virtual modules and no DOM; type-only imports (e.g. from `-api`) are fine.
- **Vitest projects (C1 §4, A4):** `unit:tsshowcase-base-vite` (Node; excludes `src/runtime/**`; hosts the tests that cross both halves: `selectTheme` against the serialised inline script, the store's and the runtime's `applyTestUpdate` agreeing) and **`unit:tsshowcase-base-vite:runtime`**: `environment: 'jsdom'`, `include` and `typecheck.include` following C1 §4's test globs under `src/runtime/`, `typecheck.tsconfig` the runtime's `dom` tsconfig, the aliases above. `jsdom` is a devDependency of this package.

## 2. The module's input: `ViteApp`

```ts
import type { PluginOption, UserConfig } from 'vite';

export interface ViteApp {
  readonly entry: string; // absolute path of the module's browser entry (its app); imported after the preludes
  readonly plugins?: readonly PluginOption[]; // framework plugins (e.g. @vitejs/plugin-react, C10)
  readonly config?: UserConfig; // framework additions merged in (resolve.dedupe/alias, …); protected keys (§3) win
  readonly userConfigFile?: string; // absolute: the user's showcase Vite config (D138, root-only D145); merged before `config`
  readonly preludes?: readonly string[]; // absolute paths imported in order before the entry: document-level CSS and setup modules (C10's showcaseStylesheets, showcaseSetup; D90, D95; root-only D145)
  readonly title?: string; // <title>; default: representation.home.name
  readonly prepareData?: (representation: Representation, state: unknown) => PreparedData; // L20: what is embedded; default: the representation as given
  readonly selectDocs?: (representation: Representation, state: unknown) => readonly DocRef[]; // D210: the docs the app can show; default: every DocRef in the representation
}

// Declared in the runtime (runtime/data/PreparedData.ts, runtime/docs/EmbeddedDoc.ts) and imported here (§1).

/** What prepareData returns: an object that keeps `revision` and `tests`; JSON-serialisable (C2 principle 2), stated in prose because
 *  interfaces such as Representation have no index signature and wouldn't satisfy a JsonObject constraint (`15d` Fix 1). */
export interface PreparedData {
  readonly revision: number;
  readonly tests: Tests;
}

/** A rendered doc as embedded in the site and returned by the runtime's loadDoc (§4.2, §6). No `format` or `assets`: the app has
 *  the DocRef, and the CLI writes the assets. */
export interface EmbeddedDoc {
  readonly id: string; // DocRef.id
  readonly html: string; // RenderedDoc.html
  readonly headings: readonly DocHeading[]; // RenderedDoc.headings
}

export function buildSite(ctx: BuildContext, app: ViteApp): Promise<BuildResult>;
export function startDevServer(ctx: DevContext, app: ViteApp): Promise<DevSession>;
```

A module's `build` is then `return buildSite(ctx, { entry, plugins: [react()], … })`. C10's `-base-react` fills `ViteApp` from its options. The fields are generic, so a Vue or Svelte module could use them unchanged.

**`prepareData` (L20):**

- `-base-vite` embeds whatever `prepareData` returns. The default, the `Representation` as is, is correct but carries e.g. `PackageInfo.raw`; `-base-react` (C10) passes a function that drops what its app doesn't read.
- **Contract (S12, T10, redesign):** the result is a JSON object that keeps `revision` and `tests` (the whole `Tests` object, §4.4 of C2), because `applyTestUpdate` merges into `tests` and the live channel compares revisions (§6, §8). Everything else is the module's to keep, reshape or drop. Since D172 results are keyed by file in `Representation.tests` and nodes reference them through associations, no node shape needs to be kept for test updates.
- Showcase loaders (§4) and the selected docs (§4.2) are taken from the unprepared representation, so `prepareData` doesn't affect them.
- **`state` (`15d` QA):** both hooks receive the analysis `state` the representation was built from: `ctx.state` in `build` and for the first `dev` snapshot, then each `DataUpdate.state` (C2 §5.8); the store keeps it per snapshot. A type test asserts that `Representation` is assignable to `PreparedData`.

**`selectDocs` (D210):** the docs the app can show; only those are rendered (through `ctx.docs.render`, so C5 records them, `renderedAssets()`) and embedded, so only rendered docs and their assets reach the output (D187, E2). The default, every `DocRef` of the representation (homes and nodes), is correct for any app; `-base-react` (C10) narrows it to what its layout presents (e.g. the home docs its `homeDocs` option selects). Refs are matched by `path` and `id` against the representation; a ref not in it is an error naming the doc. A module that renders docs itself (e.g. for its own data) may still call `ctx.docs.render`; those docs' assets are then written too, but their HTML reaches the site only if the module embeds it.

**Preludes** are the user's files, so they go through Vite's normal pipeline, including the user's PostCSS. That is right: a showcase stylesheet is the user's own CSS.

**Duplicate plugins (S16, D147):** a showcase Vite config that reuses the user's app plugins often contains the same framework plugin as `app.plugins` (e.g. `@vitejs/plugin-react`). Running both would transform every file twice and inject fast refresh twice. Plugins are de-duplicated **by name**, and **the user's copy wins** (D147, revising S16): when a plugin name from the user config also appears in `app.plugins`, the module's copy is dropped with a warning naming it. The user's copy carries the options their code needs (emotion's `jsxImportSource`, babel plugins); the module's app ships prebuilt, and C10 verifies it works under a user-configured plugin-react. The plugin README documents that the showcase config runs under Vite 8 (Rolldown), so plugins written only for Vite ≤ 7 may not work.

## 3. The Vite config

### 3.1 Assembly

`-base-vite` builds an **inline config**, in this order:

1. our base;
2. `userConfigFile`, if set (D138);
   - it is loaded with Vite's `loadConfigFromFile`, using `configLoader: 'runner'` (the default `bundle` loader writes `node_modules/.vite-temp/` into the repo, S14). Loading resolves a serve config internally, which sets `NODE_ENV=development` when it is unset, so in `build` the env guard (§7) already covers loading (T4);
   - it gets a `configEnv` carrying the protected `mode` and `command`;
   - its `build.rollupOptions` is moved into `build.rolldownOptions` before merging (S1);
3. `app.config` (normalised the same way);
4. the plugins, after de-duplication (§2): the user's first, then `app.plugins`, then ours;
5. the **protected keys** (§3.2), set last.

If the user's or the module's config set a protected key, a warning names it.

### 3.2 Protected keys: assert, don't fight (S1, D146)

Plugins are trusted code and can change the config at several points after we hand it over: `config` hooks (ordered by the hook's own `order`, not by `enforce`), the `environments.client` config and `configEnvironment` hooks (applied after `config`; the legacy `build()` and dev's `optimizeDeps` read the **client environment**), Rolldown `options`/`outputOptions` hooks, or in-place mutation. Fencing all of that in isn't achievable (T2, T15). So `-base-vite` doesn't fight plugins; it makes a misconfiguration **fail loudly**:

- **Assembly:** the protected keys are written last, under `rolldownOptions`, after both names have been normalised (§3.1).
- **One check:** in `configResolved`, our plugin compares the **client environment's** resolved values with every protected key, in resolved form (e.g. `base` is `/` in serve, `publicDir: false` resolves to `''`). A difference throws an error naming the key and saying that a plugin or the showcase Vite config changed it.
- **Output checks** in `buildSite` (§7) catch what only shows in the result.

| Key | Value | Why |
| --- | ----- | --- |
| `configFile` | `false` | never auto-load the repo's own `vite.config.*` (D138); verified in the spike |
| `root` | `ctx.repoRoot` | D144: the user's PostCSS for showcase CSS, repo-relative resolution, Vite's watcher on the user's sources |
| `base` | `'./'` | inclusion rule 1 (relocatable); in `dev` it resolves to `/` |
| `appType` | `'custom'` | the default `'spa'` serves the repo's own `index.html` for every unknown URL in `dev` (spike); our middleware serves ours (§8) |
| `mode` | `'production'` / `'development'` | D94, D143 |
| `define['process.env.NODE_ENV']` | `"production"` / `"development"` | client code never sees a leftover `NODE_ENV` (D143) |
| `publicDir` | `false` | the repo's `public/` must not be copied into the site |
| `cacheDir` | `<ctx.cacheDir>/vite` | never write the repo's `node_modules/.vite` |
| `optimizeDeps.entries` | `app.entry`, the preludes, every showcase file (§4; absolute, glob-escaped: entries are glob patterns, T8) | S4: the dependency scan reads entry HTML from disk, bypassing our `load` hook; the default would crawl the repo's own `index.html` (or find no entries and force a reload on first visit). Fixed per `dev` session: a showcase added later has its dependencies discovered lazily (one reload) |
| `server.middlewareMode` | `false` | we need the HTTP server |
| `server.ws` | not `false` | the live channel (§8) |
| `server.hmr` | not `false` | theme and showcase HMR (`reloadModule` is a no-op without it, T6) |
| `customLogger`, `clearScreen` | ours (routes to the module's `Logger`), `false` | one output channel (T14) |
| `experimental.bundledDev` | `false` | it replaces the middleware stack §8 hooks into |
| `build.write` | `false` | D64 |
| `build.watch` | `null` | `build()` must return the output, not a watcher |
| `build.lib`, `build.ssr` | unset / `false` | an app build, not a library or server build |
| `build.outDir` | `<ctx.cacheDir>/vite-out`, never written | Vite needs a value; `emptyOutDir: false` |
| `build.rolldownOptions.input` | the virtual `index.html` (§5) | the only entry |
| `build.rolldownOptions.output` | a single object (an array is rejected), `format: 'es'` | several outputs would emit colliding names; the app is an ES module (T2) |
| `build.rolldownOptions.cwd` | `ctx.repoRoot` | module ids relative to the repo (spike) |
| `build.rolldownOptions.experimental.attachDebugInfo` | `'none'` | S2: no `//#region` comments, which carry paths relative to the cwd and leak the machine layout for files outside the repo (theme and module files, global or `npx` installs) |

**Defaults that the showcase Vite config may change:**

| Key | Default | Why |
| --- | ------- | --- |
| `envDir` | `false` | D139 |
| `resolve.tsconfigPaths` | `true` | D142 |
| `build.sourcemap` | `false` | rule 2: user-enabled sourcemaps carry relative machine paths in `sources` (S2); the CLI's path check doesn't catch relative paths, so C13 documents the risk |
| `build.minify`, `target`, `cssTarget` | Vite's defaults | they affect the user's code and CSS only; our CSS bypasses them (§4.1) |

**Not part of the config:**

- **The user's PostCSS** applies only to CSS that goes through Vite's CSS pipeline: showcase CSS and preludes. Theme and module CSS never reach it (§4.1, D144).
- **Abort (`15d` Fix 12):** `ctx.signal` is checked before rendering docs, between docs, and before and after the Vite call. A running render or Vite build can't be interrupted, so on an abort `buildSite` rejects with `ctx.signal.reason` once the current step settles; it never resolves with a partial site.
- **Windows (S14):** every generated import specifier and virtual id passes through Vite's `normalizePath`.

## 4. Virtual modules (the `virtual` plugin)

The ids resolve to `\0`-prefixed virtual ids; their types live in `./client`.

| Id | Exports | Notes |
| -- | ------- | ----- |
| `virtual:tsshowcase/data` | `data` (the `prepareData` result), `mode: 'build' \| 'dev'` | D77: compiled into the bundle; in `dev`, the current snapshot (§8); its `revision` is the one shared revision (D186; `0` in builds). `mode` is generated by our plugin, so the prebuilt runtime never relies on `import.meta.env` (`15d` Fix 6) |
| `virtual:tsshowcase/themes` | `themes` (`ctx.themes`), `defaultTheme`, `shellCss`, `fontsCss` | both CSS strings come from our CSS pipeline (§4.1) |
| `virtual:tsshowcase/showcases` | `showcases: Record<RepoPath, () => Promise<Record<string, unknown>>>` | D140, D195: one lazy `import('/abs/path')` per showcase file, so each showcase file is its own chunk by default (spike); the runtime picks the export (§6) |
| `virtual:tsshowcase/docs` | `docs: Record<string, () => Promise<{ default: EmbeddedDoc }>>` (build only) | D210: keyed by `DocRef.id`; one lazy `import('virtual:tsshowcase/doc/<id>')` per selected doc (resolved by our plugin to `\0tsshowcase-doc:<id>`), so each doc is its own chunk by default (§4.2); in `dev` the table is empty and docs come from the dev server |
| `virtual:tsshowcase/entry` | nothing | the HTML's only script: imports the preludes in order, then `app.entry`. It is reached as `/@tsshowcase/entry.js`, which our plugin resolves to the virtual id, the only form that works in both build and dev (spike) |

**Showcase files (one definition, `15d` Fix 9; K5, QE):** every distinct `ShowcaseAssociation.file` on any node of the representation, at any association level. A file that is both a test and a showcase counts as a test (D173) and has no `ShowcaseAssociation`, so it isn't one. The same set feeds `optimizeDeps.entries` (§3.2), the loaders and the `dev` reload rule (§8). Every showcase file is compiled, so a broken one fails `build` even if the module never presents it (QE; documented in C13).

**One chunk each is a default (`15d` QC):** with Vite's defaults every showcase file and every doc is its own chunk (verified on Vite 8.3.4, Rolldown 1.2.14; chunk names such as `assets/_tsshowcase-doc_doc-3f2a9c1e-<hash>.js` carry no paths). A showcase Vite config that turns code splitting off (`build.rolldownOptions.output.codeSplitting: false`) or groups chunks merges them; the site stays correct (only selected docs are embedded), only bigger. Not a protected key; the tests use the default config.

**Path leaks:** absolute paths appear only in import specifiers, which Rolldown resolves away. Debug comments are off (§3.2). The output check (§7) catches anything else.

### 4.1 Our own CSS pipeline (D144)

Vite applies the repo's PostCSS config to every CSS file that goes through its CSS pipeline (spike). So **theme and module CSS never enter Vite's CSS pipeline**:

- **Theme stylesheets:** `virtual:tsshowcase/themes` reads `ctx.themeStylesheets.shell` and `.fonts` with `fs`.
- **Module CSS:** the module imports its own stylesheets as `import css from './app.css?tss-css'`; our plugin resolves and loads them, and the default export is a string. The import resolves to a `\0`-prefixed id that doesn't look like CSS (`\0tss-css:<abs>.js`), because Vite's CSS plugins and dev import analysis claim any id matching `\.css(?:$|\?)` (T3).
- **The transform** (`css/`, S8):
  1. Parse the file with `postcss`. Comments, selectors and strings are never touched.
  2. Reject `@import` in module CSS, with an error naming the file. Theme CSS arrives already flattened by the CLI (C3 §5).
  3. Strip `/*# sourceMappingURL=… */` comments.
  4. Run `postcss-value-parser` on each declaration value (including `@font-face` `src`) to find `url()`s, and string URLs inside `image-set()` (T11).
  5. Leave `data:`, `http(s):`, protocol-relative and `#fragment` URLs alone.
  6. Resolve every other URL's path part against the CSS file's directory. A missing file is an error naming the CSS file and the URL.
  7. Turn each URL's path into an asset `import`, handled by Vite's asset plugin; a `?…`/`#…` suffix (`?#iefix`, `#id`) is appended to the imported URL in the emitted string, never put into the import specifier (T11).
  8. Emit the CSS as a JS string expression concatenated with those imports.
- **Result (spike):**
  - assets are emitted and resolve to `new URL(…, import.meta.url)`;
  - small assets are inlined as `data:`;
  - everything works from a sub-path;
  - the user's PostCSS doesn't touch it;
  - `dev` needs no `fs.allow` entries.
- **Plain CSS in, plain CSS out.** What a module hands to `?tss-css` must be CSS, but the module's own package build may produce it with any toolchain (Sass, Less, PostCSS). The pipeline neither minifies nor lowers the CSS: nesting ships as written (C3 §5, amended by C3-A2). Both can be added later inside this pipeline (e.g. a Lightning CSS transform), not through Vite's options, which don't apply here.
- **Watching:** the plugin calls `this.addWatchFile` for every file it reads. In `dev`, a module CSS edit becomes an HMR update of the module that imports it.

### 4.2 Rendered docs (D187, E2, D210)

How rendered docs reach the site, with C5 owning render tracking (`renderedAssets()`, `05b` Fix 28):

- **`build`:** `buildSite` calls `selectDocs` (default: every `DocRef`), de-duplicates the refs by `id`, and renders each through `ctx.docs.render`, one at a time, **before** the Vite build, checking `ctx.signal` between docs (§3 "Abort"); a rejected render fails `buildSite` with an error naming the doc. Each rendered doc becomes a virtual module (`virtual:tsshowcase/doc/<id>`, resolved to `\0tsshowcase-doc:<id>`) whose default export is its `EmbeddedDoc` (`id`, `html`, `headings`; `assets` stay out: the CLI writes them, and the HTML already points at `tss-assets/`), and `virtual:tsshowcase/docs` holds one lazy loader per doc. A doc that isn't selected is never rendered, so neither its HTML nor its assets reach the output; when `buildSite` resolves, every doc it embedded has been rendered, so C11's `renderedAssets()` (called after the module's `build` settles) is complete.
- **`dev`:** nothing is pre-rendered. The dev server answers `GET /@tsshowcase/docs/<id>.json` by rendering the doc from the **current** snapshot's `DocsAccess` (§8), if the id is among that snapshot's selected docs, with `{ revision, doc }`: the revision of the snapshot it rendered from and the `EmbeddedDoc`. Any other id gets 404, a rejected render 500 with the message (also logged). Rendering is therefore lazy: only docs the browser asks for are rendered, and after a snapshot only those are rendered again (08b §7.10). Responses carry `Cache-Control: no-store`, and `loadDoc` requests with `cache: 'no-store'`; there is no query parameter (`15d` QD).
- **Racing a snapshot (`15d` QD):** a request served from an older snapshot can answer after a newer one was applied. `loadDoc` rejects with a `StaleDocError` (exported from `./runtime`) when the response's revision is lower than the runtime's last applied revision; the app ignores it, since it re-requests the docs it shows after every `onData`. Doc ids can change between snapshots (D184 lengthens colliding ids), so in `dev` apps take ids from the latest data, and a 404 for an id the latest data no longer has is expected around a snapshot (documented).
- **Why chunks in builds and a request in `dev`:** a lazy chunk per doc keeps the initial bundle small and needs no `fetch` path rules (the URLs are Rolldown's, relative); in `dev`, doc contents change with every snapshot while a re-imported module URL would come from the browser's module map, so a plain request is simpler than versioned module ids.
- **Doc assets** are written by the CLI in `build` (C5, C11) and served from `docAssetsDir` in `dev` (§8); `-base-vite` never copies them.

## 5. `index.html` and the inline selection script

The pipeline generates the HTML.

- **It exists only virtually.** Its id is `<repoRoot>/index.html`, which the plugin supplies through `resolveId`/`load`, so a repo's own `index.html` is never read. That is verified for builds; in `dev`, the dependency scan is redirected by `optimizeDeps.entries` (§3.2), and our middleware serves the page through `server.transformIndexHtml` (§8).

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="color-scheme" content="light dark" />
    <link rel="icon" href="data:," />
    <title>{title, HTML-escaped}</title>
    <script>/* selection script, see below */</script>
  </head>
  <body>
    <script type="module" src="/@tsshowcase/entry.js"></script>
  </body>
</html>
```

- **No `<tsshowcase-app>` in the HTML:** the module creates its host element (C10 owns the host).
- **`lang="en"`** is the shell's UI language.
- **The `data:,` icon** stops a `/favicon.ico` request that would 404 on every load.
- **The selection script** implements C2 rule 4 **for `<html>` only** and sets `document.documentElement.style.colorScheme` to the selected theme's scheme. The selection order is:
  1. the stored choice, read in `try`/`catch` because storage may throw;
  2. `defaultTheme`;
  3. `prefers-color-scheme` over the theme schemes;
  4. the first theme.
- **It is generated from the same TypeScript function the runtime's `getInitialTheme()` uses**: `selectTheme(themes, defaultTheme, storageKey, storage, matchMedia)`. The function is serialised with `Function.prototype.toString` and followed by its arguments as JSON.
  - `selectTheme` lives in its own **import-free** file (no virtual modules, no `-api` constants), so the Node-side `html/` code can import and serialise it; `THEME_STORAGE_KEY` is passed in as `storageKey` (T9).
  - The JSON escapes `<` as `\u003c`, so a label containing `</script>` or `<!--` can't break the page (S10).
  - `selectTheme` is kept free of helpers and excluded from coverage instrumentation.
  - A test evaluates the serialised text with `new Function` and asserts that it picks the same theme as `getInitialTheme()` in every case.
  - No document attribute carries the result (S7). The runtime re-runs `selectTheme` on the same inputs.
- **Inline script and CSP:** static hosts like GitHub Pages send no CSP. A site served with a strict CSP needs a hash for this script. This is documented in C13; v1 has no nonce support.

## 6. The browser runtime (`./runtime`, H28)

The runtime is framework-agnostic and made of plain functions. Importing it has no global side effects beyond reading the virtual modules.

```ts
// data/ and live/
export function getData<T extends PreparedData = Representation>(): T; // the embedded (prepared) data
export function onData<T extends PreparedData = Representation>(listener: (data: T) => void): Unsubscribe; // dev only; a no-op subscription in builds; data.revision is the message's revision
export function onTestUpdate(listener: (update: TestsUpdate) => void): Unsubscribe; // dev only
export function applyTestUpdate<T extends { readonly revision: number; readonly tests: Tests }>(data: T, update: TestsUpdate): T; // D141, D186: pure; merges update into data.tests by C2 §4.4's table and sets data.revision to update.revision; also used server-side (§8)

// docs/
export function loadDoc(id: string): Promise<EmbeddedDoc>; // D210: build: the doc's chunk; dev: the dev server renders it from the current snapshot (path chosen by `mode`, §4); rejects with a clear error for an id that isn't selected, and in dev with StaleDocError for an answer older than the last applied revision (§4.2)
export class StaleDocError extends Error {} // dev only: a doc answer from an older snapshot; apps ignore it

// showcases/
export function loadShowcase(file: RepoPath, exportName: string): Promise<unknown>; // D140, D195: the showcasing component `exportName` ('default' for the default export) of that file; rejects with a clear error for an unknown file or a missing export

// stylesheets/
export function adoptShellStylesheets(root: ShadowRoot, moduleCss: readonly string[]): void;
export function installFontsStylesheet(): void; // a <style> element in <head> (idempotent)

// theme/
export interface ThemeInfo { readonly name: string; readonly label: string; readonly scheme: 'light' | 'dark' }
export const themes: readonly ThemeInfo[];
export function getInitialTheme(): string; // rule 4; agrees with the inline script (§5)
export function getTheme(): string;
export function setTheme(name: string): void; // persists under THEME_STORAGE_KEY (in try/catch), sets <html> color-scheme, notifies subscribers
export function subscribeTheme(listener: (name: string) => void): Unsubscribe;

// document/
export function syncDocument(shell: Element): void; // <html> color-scheme from the active theme's scheme, canvas background from the shell's computed --tss-surface (H6)
```

**`applyTestUpdate`** lives in an import-free file (no virtual modules, no DOM), like `selectTheme` (§5), so the dev server's Node side imports the same function and the browser and the server merge identically (C2 §4.4).

**`adoptShellStylesheets(root, moduleCss)`** sets the root's sheets to `[themeShell, ...module]`. If constructable stylesheets are unavailable (feature-detected), it uses `<style>` elements in the same order instead, never mixing the two (rule 3). Every browser that can render the theme CSS (nesting: Safari 17.2, Chrome 120, Firefox 117) supports constructable stylesheets, so the fallback exists for test environments such as jsdom (our runtime tests, C10's smoke render); it is kept because it costs next to nothing (T18).

- **One shared theme sheet:** a single `CSSStyleSheet` holds the theme shell CSS and is shared by every shadow root, so one `replaceSync()` updates all of them. With the `<style>` fallback, each root gets its own element, and all of them are tracked.
- **A repeated call for the same root replaces the module sheets and keeps the theme sheet.** Modules call it again from their HMR handler when their CSS changes (§4.1).

**The fonts stylesheet** is a `<style>` element in `<head>`, not an adopted document sheet. Libraries that assign `document.adoptedStyleSheets = […]` would otherwise drop it (S14).

**Theme stylesheet updates in `dev`:**

- The adapter (`runtime/internal/virtual.ts`, §1) accepts HMR updates of `virtual:tsshowcase/themes` (§8) and hands the new CSS to the theme stylesheet code.
- On an update, it calls `replaceSync()` on the shared theme sheet (or updates every tracked `<style>`) and replaces the fonts `<style>` text.
- A module calls `adoptShellStylesheets` once and gets these updates for free.

**Showcase loaders in `dev` (S5):**

- The adapter accepts HMR updates of `virtual:tsshowcase/showcases` and hands them to the showcases code. In the accept callback: if the set of showcase files changed, it swaps its loader table; otherwise it calls **`location.reload()`** (T1).
- The second case is an edit to a showcase file that isn't a framework refresh boundary (e.g. a React showcase file that also exports a non-component, such as a flag) or to a plain helper, which propagates up to the table. `import.meta.hot.invalidate()` can't be used: the server ignores it unless the module accepts itself, and it propagates to importers rather than reloading.
- Edits handled by framework HMR (React refresh, C10) never reach it.
- A change to a file's showcasing components (an export added or removed) changes only the data: the app sees it through `onData`, and `loadShowcase` reads the export from the freshly imported module.

**Live updates (D186, redesign):**

- **One revision.** Data snapshots and test updates share one monotonic revision (C2 §4.4). The runtime keeps a single "last applied" revision, starting with the embedded `data.revision`, and drops any `tsshowcase:data` or `tsshowcase:tests` message whose revision isn't higher; it keeps no data copy (D141, T5).
- **Nothing is consumed before a listener exists (`15d` Fix 7):** the runtime ignores live messages until the first listener subscribes, and a message advances the "last applied" revision only when it is delivered to at least one listener of its stream (`onData` for `tsshowcase:data`, `onTestUpdate` for `tsshowcase:tests`). So a snapshot that arrives during an async mount isn't lost: the hello recovers it.
- **Hello.** In a microtask after the **first listener** subscribes (`onData` or `onTestUpdate`), the runtime sends `tsshowcase:hello { revision }` with its last applied revision. If the server's snapshot is newer, it replies to that client with one `tsshowcase:data` message carrying the current prepared snapshot, which already includes every test update (§8). Apps subscribe to both streams in the same tick (documented), so the reply reaches `onData` whichever they subscribe first; an app that renders test results from updates also subscribes to `onData` (any app needs it for structure changes anyway). There is no hello on reconnect: Vite's client reloads the page after a lost connection, which re-imports the data module (`15d` Fix 8).
- **Docs in `dev`.** `loadDoc(id)` requests `/@tsshowcase/docs/<id>.json` with `cache: 'no-store'` (§4.2); after an `onData` the app calls it again for the docs it shows. In builds it calls the doc's lazy loader.

**`syncDocument` is the module's duty (S6):**

- `setTheme` only persists the choice, sets `<html>` `color-scheme` and notifies subscribers.
- The module renders `data-theme` from `subscribeTheme`/`getTheme` (C10, H28), and calls `syncDocument(shell)` **after** the new `data-theme` is applied (in React, a layout effect), because the computed `--tss-surface` only reflects the new theme then.
- The runtime never calls it on its own.

**`THEME_STORAGE_KEY` is shared per origin (L19).** Every GitHub Pages project site of one user shares the stored choice. That is harmless, because a stored name that isn't among the included themes falls through (rule 4). It is documented in the runtime README and in C13.

## 7. `buildSite`

1. Check `ctx.signal`.
2. **Render the selected docs** (§4.2) and collect the showcase files (§4), both from the unprepared `ctx.representation`; compute `prepareData(ctx.representation, ctx.state)`.
3. **Env guard (D94, spike):** set `process.env.NODE_ENV = 'production'` for the whole of config assembly (including loading the showcase Vite config, T4) and the `vite.build()` call, and restore it in `finally`.
   - When the variable was unset before, restoring means **deleting** it, because Vite leaves it set.
   - The guard is needed because Vite derives `isProduction` and `import.meta.env.PROD` from `NODE_ENV`, not from `mode`.
   - It is safe in `build`, because no watcher runs in that process and no test run overlaps the module's `build`: the CLI finishes running tests first and never runs other process-state work alongside (C2 §6, C7 §7, C11; `07b` QC).
4. Call `vite.build(config)` with `write: false`. The result is a single `{ output }`, since one output is protected (§3.2).
5. **Map the output to `BuildFile[]`:**
   - chunk → `{ path: fileName, contents: code }`;
   - asset → `{ path: fileName, contents: source }`, where `source` is a string or `Uint8Array`;
   - POSIX paths, sorted.
6. **Output checks** (D146; the CLI checks too, C11): a single ES output, `index.html` at the root, no file under `tss-assets/` (rule 1, `DOC_ASSETS_DIR`), and no leaked build-machine paths (rule 2). A violation throws, naming the file and the string.
   - **What the path check looks for (`15d` QB):** it is a backstop against the toolchain, a module or the CLI writing this machine's layout into the site (debug comments, sourcemaps, surviving module ids, data or test errors carrying paths), not a judgement of user content. Its candidates come **only from paths the build itself handled**: `ctx.repoRoot` and its real path, `ctx.cacheDir`, the directories of the entry, the preludes, the theme stylesheets, the showcase files and the showcase Vite config, and the directories of the absolute module ids Rolldown reports for the bundle; each also in its form relative to the process cwd. Nothing else on the machine is inspected, and generic shapes (`/usr/…`, `../`) are never searched for: bundled code and READMEs are full of them.
   - **Excused:** an occurrence whose string also appears in the raw source (`ctx.docs.read`) of a doc rendered in this build; the author wrote it, so it is content. Every other occurrence is a leak.
   - The output itself is the same on every machine; only the searched strings differ, and a real leak is a leak everywhere. C11's own check uses the same rule (its excuse list: the docs behind `renderedAssets()`).
7. Return `{ files }`.

## 8. `startDevServer`

- **`createServer`** with the §3 config:
  - `server.host` and `server.port` come from `ctx.host`/`ctx.port` (D113);
  - `strictPort: true` when a port is given, so the server fails clearly instead of moving to another port;
  - `DevSession.url` is `server.resolvedUrls.local[0]`, falling back to `network[0]` (S14).
- **Exposure (S11):** Vite serves any file under the root that `fs.deny` doesn't exclude, gitignored files included. When the **resolved** `server.host` isn't loopback (from `--host` or the showcase Vite config, T6), a warning says so; C13 documents it. `fs.deny` stays at Vite's default (its secrets patterns).
- **The environment (D143, as reworded by S3):**
  - `-base-vite` never sets or restores `process.env` in `dev`; `mode` and the `define` are explicit;
  - Vite itself sets `NODE_ENV=development` when it is unset, consistent with D137;
  - under `NODE_ENV=production` a warning says that Vite reports production to plugins (React fast refresh is disabled) and that pre-bundled dependencies are production builds.
- **Middlewares,** registered directly in `configureServer`, so they run before Vite's own:
  - `/` and `/index.html` serve our HTML through `server.transformIndexHtml`. With `appType: 'custom'`, unknown paths give 404, and the repo's own HTML is never served (spike).
  - `/@tsshowcase/docs/<id>.json` renders a selected doc from the current snapshot (§4.2); GET/HEAD only.
  - `tss-assets/` (D125; in `dev` it holds the assets of **every** `DocRef`, pre-copied by the CLI before each `onDataUpdate`, 02b Q2) serves `ctx.docAssetsDir` with `sirv`: `dev: true` (no file list cached at start, S14), GET/HEAD only, no directory listing, no dotfiles. Every other request under `tss-assets/` gets 404 and never falls through to Vite.
- **`server.fs.allow`:** Vite's default, with no additions. Module files outside the root, the theme stylesheets and their assets all load through the module graph and our CSS pipeline (spike).
- **Snapshot store and live channel (S9, redesign):** `-base-vite` keeps the current **unprepared** snapshot: the `Representation` (with every test update merged in), the `DocsAccess` and `state` it came with, and its selected docs (`selectDocs`). It starts from `ctx.representation`, `ctx.docs` and `ctx.state` (`15d` Fix 10). Both streams come from the CLI and share one monotonic revision (D186), so the store's revision is simply the snapshot's `revision`, raised by each applied test update.
  - **`ctx.onDataUpdate(update)`:**
    1. replace the snapshot with `update.representation`, `update.docs` and `update.state` (the previous `DocsAccess` is not used again, C2 §5.8), and re-run `selectDocs(representation, state)`;
    2. send `tsshowcase:data` `{ revision, data: prepareData(representation, state) }` over `server.ws.send`;
    3. invalidate `virtual:tsshowcase/data` **without** an HMR update, so the next page load gets the fresh data;
    4. if the set of showcase files (§4) changed, reload `virtual:tsshowcase/showcases` **with** an HMR update (§6).
  - **`ctx.onTestUpdate(update)`:**
    1. apply it to the snapshot server-side with `applyTestUpdate` (merging into `tests`, raising the revision);
    2. send `tsshowcase:tests` with the update;
    3. invalidate the data module (no HMR).

    A page reload then shows current test results.
  - **A throwing hook (`15d` Fix 11):** if `selectDocs` or `prepareData` throws (or `selectDocs` names an unknown ref) for a new snapshot, the error is logged through the module's logger and the previous snapshot stays in place, revision unchanged; a later update replaces it. A throwing `prepareData` while answering a hello or loading the data module is logged too, and that request gets no data (the page keeps what it has).
  - **Messages are dropped by revision, not reordered:** a data update or test update whose revision isn't higher than the store's is ignored with a debug log. The CLI guarantees that every `onDataUpdate` snapshot already includes the test updates it forwarded before and carries a higher revision (C11, T12), so nothing regresses.
  - **`ctx.onThemeStylesheetsUpdate`:** `server.environments.client.reloadModule` on `virtual:tsshowcase/themes`. The plugin re-reads the files the CLI rewrote, and the runtime applies the result (§6). The pushed contents aren't used, because the files are the source of truth (C2 B8).
  - **Hello:** a page could miss updates sent between loading the data module and the app subscribing (an async mount). So the runtime sends `tsshowcase:hello` `{ revision }` after the **first listener subscribes** (to either stream), with its last applied revision (the embedded one at first); see §6 for the delivery rules. If the store's revision is higher, the server replies to that client alone (`server.ws.on('tsshowcase:hello', (data, client) => client.send(…))`) with one `tsshowcase:data` message carrying the current prepared snapshot. A hello sent before the socket opens waits for the connection in Vite's client. No reconnect hello: Vite's client reloads the page after a lost connection.
  - **Stale messages:** the runtime drops any message whose revision isn't higher than the last one it applied (one counter across both streams, D186).
  - **HMR calls:** only the environment API (`server.environments.client.reloadModule`, `moduleGraph.invalidateModule`); the deprecated `server.reloadModule` crashes (spike).
  - **Ordering:** every `ctx.on…` subscription is made **before** `listen()` (S14). Updates that arrive before the server is up only update the snapshot.
- **`DevSession.closed`:**
  - resolves when `close()` finishes;
  - rejects when the HTTP server emits `error` after start.

  Every subscription is removed on close.

## 9. Tests

### Node (Vitest, `unit:tsshowcase-base-vite`)

**Config assembly:**

- the merge order;
- the `rollupOptions` → `rolldownOptions` normalisation;
- plugin de-duplication: the user's copy wins, with a warning (D147);
- the config loader writes nothing into the fixture.

**Protected keys (D146):** for each row of §3.2, a user config and a plugin hook (`config` with `order: 'post'`, `configEnvironment`) changing the value; the check throws, naming the key. The output checks catch a changed output (`outputOptions` hook).

**A real `buildSite`** of a small fixture (`fixtures/base-vite-basic`, with the stub entry and theme files outside the fixture root at a different depth, and a `postcss.config.js` that leaves a marker). It checks:

- **HTML:** the `index.html` content; the fixture's own `index.html`, `vite.config.ts` and `public/` are unused.
- **Data:** the embedded data (the default `prepareData` and a narrowing one that keeps `revision` and `tests`).
- **Docs (D210):** with the default `selectDocs` every `DocRef` is rendered and embedded as its own chunk; with a narrowing `selectDocs`, an unselected doc's HTML is absent from every output file and its `render` was never called (a spy `DocsAccess`); a ref not in the representation and a rejecting `render` fail `buildSite`, naming the doc; an abort between renders rejects with `ctx.signal.reason` before the Vite build; `selectDocs` and `prepareData` receive `ctx.state`.
- **CSS:**
  - theme CSS with a font, a large image and a small image (emitted, inlined, relative URLs), plus module CSS via `?tss-css` (not claimed by Vite's CSS plugins, T3), both **without** the PostCSS marker, while the showcase CSS has it;
  - `?#iefix` suffixes and `image-set()` string URLs;
  - `@import` in module CSS is rejected;
  - a missing `url()` target is an error;
  - `sourceMappingURL` comments are stripped;
  - `url(` in comments and selectors is untouched.
- **Chunks:** one chunk per showcase file (several showcasing components in one file share it; a weakly associated showcase is included too) and one per selected doc.
- **Paths (QB):** no leaked build-machine path in any file, also with `minify: false`; a README quoting `/etc/`, `../../utils` and the fixture's real absolute root passes (excused by the raw doc source), while the same root injected through `prepareData` fails, naming the file and the string.
- **Doc assets:** no files under `tss-assets/`.
- **Environment:** `NODE_ENV` is restored afterwards (also when it was unset, with a `userConfigFile`, T4, and after a failing build).

**D94:** with `NODE_ENV=development` set before `buildSite`, the result is still a production build (`import.meta.env.PROD`, the `NODE_ENV` replacement).

**`startDevServer`:**

- **Pages:** serves `index.html`; unknown paths get 404.
- **Dependency scan:** a readable repo `index.html` that imports a missing module doesn't disturb startup (S4); a repo path with glob characters still scans (T8).
- **Module CSS** via `?tss-css` is served as JS and hot-updates its importer (T3).
- **Docs (D210, QD):** `/@tsshowcase/docs/<id>.json` answers `{ revision, doc }` from the current snapshot (after an `onDataUpdate`, the new contents and revision), only for selected ids (others 404), only when requested (a spy `DocsAccess`), 500 on a rejected render; `Cache-Control: no-store`.
- **`tss-assets/`:** serves only files from `docAssetsDir`, including files added after start; traversal attempts and misses get 404.
- **Server:** `strictPort`; the non-loopback warning, also for a host set by the showcase Vite config (T6).
- **Live channel:**
  - events reach a `ws` test client in order, with the one shared revision (D186);
  - a test update followed by a reload of the data module shows the new results and the raised revision;
  - a data or test update with a revision not higher than the store's is ignored;
  - a `hello` with an older revision gets one `tsshowcase:data` reply (to that client only) with the current snapshot, test updates included; a `hello` with the current revision gets none (T5);
  - after an `onDataUpdate`, the previous `DocsAccess` is never called again; the new `state` reaches `selectDocs` and `prepareData`;
  - a throwing `selectDocs`/`prepareData` keeps the previous snapshot and revision, logged (Fix 11);
  - the hello reply goes through `server.ws.on` to the asking client only;
  - `applyTestUpdate` in the store and in the runtime give the same result on the same sequence (the shared file, Fix 3);
  - a theme stylesheet update sends an HMR update for the themes module;
  - the showcases module is reloaded only when the file set changes; an edit to a showcase that isn't a refresh boundary reloads the page (T1).
- **Environment:** `-base-vite` sets nothing in `process.env`; a warning appears under `NODE_ENV=production`.
- **Shutdown:** `close()` releases the port.

### Runtime (Vitest, jsdom, S13)

**Each runtime exporting file has its test named after it (C1 §8, `15d` Fix 13)**: `getData`, `onData`, `onTestUpdate`, `applyTestUpdate`, `loadDoc`, `loadShowcase`, `adoptShellStylesheets`, `installFontsStylesheet`, `getInitialTheme`, `getTheme`, `setTheme`, `subscribeTheme`, `syncDocument` (one file per concept, or one test per file where a file holds several). The adapter is mocked (K7, Fix 4). Cases:

- `getInitialTheme` agreeing with `selectTheme` (the inline-script comparison itself runs in the Node project, §1);
- `setTheme`, `subscribeTheme` and storage failures;
- `applyTestUpdate`: every row of C2 §4.4's merge table, the revision raised, purity (the input is unchanged);
- the live revision counter: one counter across `onData` and `onTestUpdate`, stale messages dropped, messages before the first listener ignored without advancing it, the hello sent once in a microtask after the first subscription, and the hello reply reaching `onData` whichever stream is subscribed first in the same tick (Fix 7);
- `loadDoc`: the build table and the `dev` request (`cache: 'no-store'`, path chosen by `mode`), `StaleDocError` for an older answer, a rejection for an unknown id;
- `installFontsStylesheet` idempotence; `syncDocument` from a shell's computed `--tss-surface`; `getData` and `onData` (a no-op in builds);
- `loadShowcase(file, exportName)`: a named and the default export, rejections for an unknown file and a missing export;
- the `<style>` fallback: order, re-adoption replacing module sheets only, and updates;
- the showcases module's accept rule (swap or `location.reload()`).

Constructable stylesheets in a real browser were covered by the spikes; as acceptance, they are covered by C12.

### README samples

Type-checked (D56).

## 10. Documentation

**Package README:**

- what `-base-vite` does;
- a minimal non-React module using it (a type-checked sample);
- the `ViteApp` fields, the `prepareData` contract (keep `revision` and `tests`) and `selectDocs` (only selected docs are rendered and embedded, D210);
- the protected and defaulted keys, and why;
- the showcase Vite config as a module author passes it on: root-only (D145), runs under Vite 8, plugin de-duplication;
- `?tss-css` and its rules: plain CSS, no `@import`, not minified;
- the `state` argument of the hooks; one chunk per doc and per showcase as a default (QC); the output path check and its doc-source excuse (QB).

**Runtime README:**

- the API;
- the order of calls: `installFontsStylesheet` → `adoptShellStylesheets` → render `data-theme` from `getTheme` → `syncDocument` after each applied theme;
- HMR of module CSS;
- the showcases reload rule;
- `loadShowcase(file, exportName)` and `loadDoc(id)`, and re-loading the shown docs after `onData` in `dev`; `StaleDocError`; doc ids taken from the latest data (they can change, D184);
- the one shared revision, the hello, subscribing to both streams in the same tick, and why an app rendering test results subscribes to `onData` too;
- the per-origin storage note.

## Acceptance criteria

1. A stub module (test-only) built with `buildSite` produces a site that loads from a sub-path, embeds the data and the selected docs (one chunk each, no unselected doc rendered or embedded) and adopts the theme shell sheet. Here this is covered by the Node-side assertions of §9; the headless check is C12's.
2. Changing any protected key of §3.2 through a user config or a plugin hook fails with an error naming the key, verified by a test per key.
3. Theme and module CSS never pass through the repo's PostCSS; showcase CSS does.
4. No build-machine path (as defined in §7: from the build's own inputs, excused only by a rendered doc's raw source) appears in the output, minified or not.
5. `startDevServer`:
   - serves the app;
   - keeps a snapshot that survives reloads, including test results;
   - delivers both live streams with the one shared revision and the hello;
   - renders docs on request from the current snapshot, selected ones only, tagged with its revision;
   - sends theme HMR updates;
   - serves `tss-assets/` from `docAssetsDir` only;
   - never sets `process.env`;
   - stops cleanly.
6. `selectTheme` and the inline script agree in every case.
7. There is no React, or any other framework, anywhere in the package (enforced by a `no-restricted-imports` rule).

## Recorded for later chunks

- **C6 (amendment, applied):** see "Amendments" below (D145, D148).
- **C8 (`15d` G17):** whether ambient `declare module` declarations (e.g. `src/client/client.d.ts`) are symbols, and how a declaration-only `exports` subpath (`./client`) fits C1 §8's "entry files only re-export".
- **C8:** `-base-react`'s options schema declares `viteConfig` (D138) as a path option (`OPTION_FORMAT.path`, D116). Options are root-only by default (D148): `viteConfig`, `showcaseStylesheets` and `showcaseSetup` stay unmarked (there is no `testRunner` option since D172) (their descriptions say "root config only"), and only genuinely per-package options (e.g. analysis or rule presets) get `"x-tsshowcase-per-package": true`.
- **C10:**
  - **`ViteApp`:** fill it with plugin-react, React dedupe and resolution, a `prepareData` that drops unread fields but keeps `revision` and `tests`, a `selectDocs` for the docs its layout presents (incl. the `homeDocs` selection, D185), `preludes` from `showcaseStylesheets`/`showcaseSetup`, and `userConfigFile` from `viteConfig`; verify the app works when the user's own plugin-react wins de-duplication (D147);
  - **CSS:** import the module's CSS as `?tss-css` (plain CSS, no `@import`; the `-lint` module preset applies), and re-adopt module CSS on HMR;
  - **Runtime calls:** create the host and call the runtime in the documented order; call `syncDocument` in a layout effect after `data-theme` changes (S6);
  - **Showcases:** load them through `loadShowcase(file, exportName)` (D195). A showcase file exporting a non-component next to its showcasing components (e.g. a `shell` flag, if C8's format keeps one) isn't a refresh boundary, so edits to it reload the page (S5).
  - **Docs:** load the docs a page shows through `loadDoc(id)`, and again after `onData` in `dev`, ignoring `StaleDocError`; take doc ids from the latest data; subscribe to `onData` and `onTestUpdate` in the same tick and apply them (with `applyTestUpdate`) under the one revision; `selectDocs`/`prepareData` may use the analysis `state` (QA).
- **C11:**
  - passes `cacheDir` per module;
  - rebuilds `docAssetsDir` (every `DocRef`'s assets) before each `onDataUpdate`;
  - calls `renderedAssets()` after the module's `build` settles; `buildSite` has rendered every doc it embeds by then (§4.2);
  - runs tests to completion before the module's `build` (no test run overlaps `vite.build`, `07b` QC);
  - handles a rejected `build` (a failed doc render, an output check, an abort with `signal.reason`);
  - its own output path check uses §7's rule: candidates from the build's own inputs only, excused by the raw source of rendered docs (`15d` QB);
  - passes each snapshot's `state` in `DataUpdate` (QA);
  - rewrites the theme stylesheet files before calling `onThemeStylesheetsUpdate` (the files are what `-base-vite` reads);
  - changing which themes are configured, or the showcase Vite config (or a file it imports), needs a restart of `dev`;
  - every `onDataUpdate` snapshot includes all test updates already forwarded through `onTestUpdate` and carries a higher revision than any of them (one shared revision, D186), so the dev server's snapshot never regresses (T12).
- **C12:** in a real browser (Chromium locally):
  - constructable-sheet adoption;
  - theme asset URLs from a sub-path;
  - theme HMR, the live channel and the hello;
  - doc chunks from a sub-path and docs re-loaded after a `dev` snapshot;
  - module CSS bypassing PostCSS;
  - `hot.accept` from the published `dist/runtime` inside `node_modules` (S5).
- **C13:**
  - the showcase Vite config (root-only, Vite 8, plugin de-duplication where the user's copy wins, the protected keys and the error when one is changed);
  - no `.env` by default (D139);
  - the user's PostCSS applies to showcase CSS and showcase stylesheets only;
  - user-enabled sourcemaps leak relative machine paths;
  - the dev server exposes the repo (gitignored files included) on a non-loopback host;
  - CSP and the inline script;
  - per-origin theme storage (L19);
  - `dev` under `NODE_ENV=production` (D143);
  - every showcase file in scope is compiled, weak ones included: fix or remove stray ones (QE);
  - one chunk per doc and per showcase is the default; turning code splitting off still works (QC);
  - in `dev`, doc ids can change between snapshots (D184).
- **Later (not v1):** minification or lowering inside our CSS pipeline, if needed.

## Out of scope

- React and plugin-react (C10);
- the host element and the isolation duties (C10);
- writing the site (C11);
- the CLI cache (C11);
- scoping theme CSS (C3);
- browser acceptance (C12).

## Amendments (applied to the documents, approved together with this plan, 2026-10-19)

- **C2 amendment B8 (S7, T17):**
  - §5.8, last bullet: "the CLI regenerates both `themeStylesheets` in place and pushes the new contents through `onThemeStylesheetsUpdate`; the module applies them" becomes "…; the module re-reads the files or applies the pushed contents (`-base-vite` re-reads the files and hot-reloads them, C15)".
  - §8 rule 3: "in `dev`, applies `onThemeStylesheetsUpdate` contents" becomes "in `dev`, applies the updated stylesheets after `onThemeStylesheetsUpdate`".
  - §5.11: **per-package options** (D148): module options are root-only by default; a top-level options property opts into per-package values with the schema keyword `"x-tsshowcase-per-package": true` (`PER_PACKAGE_OPTION_KEYWORD`, exported from `module/` next to `OPTION_FORMAT`), beside the property's ordinary keywords; the CLI merges only marked keys from package configs and warns about and drops the rest.
- **C3 amendment C3-A2 (wording, S7, D144):**
  - §5 "Handing over": "`-base-vite` forwards them over its live channel and calls `replaceSync()` on the adopted sheets. The dev server must be allowed to read the cache and theme asset directories (`server.fs.allow`, C15)" becomes "`-base-vite` re-reads the files, re-processes them with its own CSS pipeline and hot-reloads them into the adopted sheets (`replaceSync()`); no `fs.allow` entries are needed (C15)".
  - §5 "Browser support": "a toolchain that lowers nesting for older targets (Vite's default `cssTarget` in builds) produces equivalent flat CSS" becomes "`-base-vite` ships theme and module CSS as written, without lowering (D144); a toolchain that lowers nesting produces equivalent flat CSS".
- **C6 amendment (D145, D148):** §7 replaces the hard-coded `testRunner` rule: module options are root-only by default; from a package config's `moduleOptions`, only top-level options the module's schema marks with `"x-tsshowcase-per-package": true` are merged, and any other key is warned about ("`<key>` can only be set in the root config") and dropped before merging and validation; the keyword is registered with ajv as an annotation. No option names are hard-coded in the CLI.
- **Super-plan R9 (C15 planning, D138–D148, spike `15a`, reviews `15b`, `15c`):** C15 is updated:
  - our own CSS pipeline for theme and module CSS (`?tss-css`) replaces "`?inline`-style" (D144);
  - no `fs.allow` additions;
  - the showcase Vite config option (D138) and `.env` off (D139);
  - showcase loaders (D140);
  - the runtime's role in test updates (D141);
  - `resolve.tsconfigPaths` (D142);
  - `dev` never touches `process.env` (D143);
  - protected config keys, asserted rather than enforced (D146);
  - plugin de-duplication, the user's copy wins (D147);
  - the dev snapshot survives reloads.

  C10 gains `syncDocument` after `data-theme`, `?tss-css` and the `viteConfig` option. C11 gains the restart rule for the showcase Vite config. C2 bullet: B8. C3: C3-A2. C6: module options root-only by default, per-package ones marked in the schema (D145, D148).
- **AGENTS:**
  - D138–D148 in the log (recorded during planning);
  - open points: the C15 row removed;
  - C8, C10, C11, C12, C13 rows extended from "Recorded for later chunks";
  - C2, C3 and C6 tracker rows note their amendments;
  - C15 marked approved.

## Redesign amendment (plan 9 of the representation redesign's application; approved 2026-10-21)

Plan 9 of the redesign's application (`08b-representation-redesign.md` §7.9, §7.10, §12; `08e` E2, E7), after AGENTS, C2, C1, C3, C4, C5, C6 and C7; choices and review in `15d-c15-redesign-review.md`. The pipeline, the config, the CSS pipeline, the HTML and the theme runtime are unchanged; what changes is the data it embeds and delivers:

- **§1:** the dogfood layout (C1 §8, A4): exported areas `pipeline/` and the runtime's areas with READMEs, one file per concept, tests named after the files; `client.d.ts` in its own area; the runtime's virtual-module imports confined to `runtime/internal/virtual.ts`, stubbed through aliases to `test/virtual/` outside the scope (`01e`); the `unit:tsshowcase-base-vite:runtime` project (jsdom).
- **§2:** `prepareData(Representation) → PreparedData` keeping `revision` and `tests` (replacing the hierarchy-shape contract); `title` defaults to `home.name`; `selectDocs` (D210); `EmbeddedDoc`.
- **§4:** `virtual:tsshowcase/data` exports `data` only (its `revision` is the shared one); showcase loaders per file with showcasing components, any association level (D195); `virtual:tsshowcase/docs` (D210). **§4.2 (new):** docs rendered before the Vite build and embedded as one chunk each; in `dev` rendered on request from the current snapshot.
- **§6:** `getData`/`onData` over `PreparedData`; `onTestUpdate` and `applyTestUpdate` over `TestsUpdate` and `Tests`, raising the revision; `loadDoc(id)`; `loadShowcase(file, exportName)`; one revision counter and one hello (D186).
- **§7:** docs rendered first; no test run overlaps `vite.build` (`07b` QC).
- **§8:** the docs endpoint; `docAssetsDir` holds every `DocRef`'s assets (02b Q2); the snapshot store holds the representation, its `DocsAccess` and selected docs under the one revision; stale updates ignored; the hello answered with one data message (the "two producers" are both the CLI now, 02b).
- **Review (`15d-c15-redesign-review.md`, two zero-context agents):** questions QA–QE settled (QA: the hooks get `state`; QB: the output path check takes candidates only from the build's own inputs and excuses strings found in a rendered doc's raw source; QC: one chunk each is a default; QD: dev doc answers tagged with their revision, `StaleDocError`, no query parameter; QE: broken weak showcases fail `build`, confirmed) and Fix 1–16 applied (`PreparedData` as an interface; runtime-side types; two TS projects; the HMR adapter; `./client` copied; `mode`; delivery rules and the hello; no reconnect hello; one showcase-file definition; the initial store; throwing hooks; abort; tests per file; small gaps; a citation; AGENTS rows).
- **§9, §10, acceptance criteria, recorded points:** tests and docs for the above; C10 gains `selectDocs`, `loadDoc`, `loadShowcase(file, exportName)`; C11 gains the render-before-resolve guarantee for `renderedAssets()`, the test-run ordering and the revision rule; C12 gains doc chunks and dev doc reloads.
- **C2 (wording):** §8's embedding note points at C15 §4.2.
- **Unchanged:** §3 (apart from `optimizeDeps.entries` naming showcase files), §4.1, §5, the theme, stylesheet and document parts of §6, the env guard and output checks of §7, the server, exposure, environment and middleware rules of §8.

## Open questions

- ✅ **Q1 Vite `root` and our CSS (D144):** the root is the repo; theme and module CSS go through our own pipeline (§4.1).
- ✅ **Q2 The showcased repo's bundler config (K18; D138):** a module option pointing at a showcase Vite config, merged with `mergeConfig`; the protected keys win; the repo's own `vite.config.*` is never auto-loaded. Root-only (D145).
- ✅ **Q3 `.env` files (D139):** off by default (`envDir: false`); the showcase Vite config may opt in.
- ✅ **Q4 Showcase loaders (D140):** generated by `-base-vite`; a framework base may still bring its own through `ViteApp.plugins`.
- ✅ **Q5 Test updates (D141):** the runtime forwards them and offers the pure `applyTestUpdate`; the dev server also applies them to its snapshot (S9).
- ✅ **Q6 Pipeline spike:** done, `15a-c15-spike-results.md`.
- ✅ **Q7 `resolve.tsconfigPaths` (D142):** on by default.
- ✅ **Q8 `NODE_ENV` in `dev` (D143, reworded by S3):** `-base-vite` never sets or restores it; explicit `mode` and `define`; a warning under `NODE_ENV=production`.
- ✅ **Review `15b` (S1–S16):** applied, with S15 decided as D145.
- ✅ **Review `15c` (T1–T18):** applied, with T15 → D146, T16 → D147, T17 → D148, T18 keep the fallback.
