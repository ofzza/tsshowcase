# Chunk plan C15: Vite pipeline helper (`@ofzza/tsshowcase-base-vite`)

> Status: **APPROVED** (round 4, 2026-10-19), together with C2 amendment B8, C3 amendment C3-A2, the C6 amendment and super-plan R9, all applied.
>
> - All open questions are resolved (Q1–Q8: D138–D144).
> - The pipeline spike (`15a-c15-spike-results.md`) is applied.
> - The first fresh-eyes review (`15b-c15-plan-review.md`, S1–S16) is applied, with the user's decisions (S15: D145; S3 rewords D143).
> - The second review (`15c-c15-plan-review.md`, T1–T18) is applied, with the user's decisions (T15: D146; T16: D147; T17: D148; T18: keep the `<style>` fallback).
> - Amendments to C2 (B8), C3 (C3-A2) and C6, plus super-plan R9, are listed at the end and applied to those documents (approved with this plan).
>
> Decisions referenced as `D<n>` are in AGENTS.md §3.

## Goal

The reusable Vite build and dev pipeline that modules call from their `build(ctx)` and `dev(ctx)` (D43), plus a small framework-agnostic **browser runtime** that their apps call (H28).

A module brings its framework plugins and its app entry. `-base-vite` does everything that is the same for every framework:

- assembling and protecting the Vite config;
- the virtual `index.html`;
- data embedding and the showcase loaders;
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
- an explicit `mode`/`NODE_ENV` (D94);
- a dev server that doesn't rely on `process.env.NODE_ENV` (C7 §7, D143).

## 1. Where the code lives

```
packages/tsshowcase-base-vite/src/
  index.ts                  Node entry: re-exports the Node units
  build-site/               exported unit: buildSite(ctx, app) → BuildResult (§3, §7)
  start-dev-server/         exported unit: startDevServer(ctx, app) → DevSession (§3, §8)
  vite-app/                 exported unit (type-only): ViteApp, the module's input (§2)
  config/                   internal: assembling, merging and protecting the Vite config (§3)
  virtual/                  internal: the virtual-modules plugin (§4)
  css/                      internal: our own CSS pipeline (§4.1)
  html/                     internal: index.html and the inline selection script (§5)
  output/                   internal: Rolldown output → BuildFile[] (§7)
  env-guard/                internal: NODE_ENV handling for builds (§7)
  dev-server/               internal: middlewares (HTML, tss-assets/), live channel, snapshot store (§8)
  runtime/                  browser entry (subpath `./runtime`), its own tsconfig with the `dom` lib (§6)
    index.ts
    data/  live/  stylesheets/  theme/  document/  showcases/
  client.d.ts               ambient declarations for the virtual modules and `?tss-css` imports (subpath `./client`, types only)
```

- **Everything lives under `src/`** (S13), so the package has one `unitRoot` (C2 §5.4). The runtime sits in its own directory with its own `dom` tsconfig.
- **Exports:**
  - `.` — Node only: the pipeline;
  - `./runtime` — browser only: imported by the module's app;
  - `./client` — types for `virtual:tsshowcase/*` and `*.css?tss-css`, referenced from a module's app tsconfig.
- **The runtime only works inside this pipeline.** It imports the virtual modules, so it runs only inside a `-base-vite` build or dev server. That is intentional: it is the runtime *of this pipeline*.
- **Runtime dependencies:**
  - `vite@^8`, a real dependency: it ships with the CLI (D4's "bundler plugins ship with the CLI/module"), and the showcased repo is never tied to Vite. Vite 8 needs Node `^20.19 || >=22.12`, which our floor (D49) satisfies;
  - `@ofzza/tsshowcase-api`;
  - `postcss` and `postcss-value-parser` (§4.1);
  - `sirv` (§8).
- **The runtime is consumed by Vite, not by Node.** `./runtime` is published as built ESM (`dist/runtime/*.js`), and Vite bundles it into the app like any other dependency. It is excluded from `optimizeDeps`, so that its virtual-module imports resolve through our plugin.
- **Dogfooding (D41):**
  - the exported units are `buildSite`, `startDevServer`, `ViteApp` and the runtime units listed in §6;
  - each has a README and tests;
  - none has a showcase: these are functions and types, for which showcases are optional (D37).

## 2. The module's input: `ViteApp`

```ts
import type { PluginOption, UserConfig } from 'vite';

export interface ViteApp {
  readonly entry: string; // absolute path of the module's browser entry (its app); imported after the preludes
  readonly plugins?: readonly PluginOption[]; // framework plugins (e.g. @vitejs/plugin-react, C10)
  readonly config?: UserConfig; // framework additions merged in (resolve.dedupe/alias, …); protected keys (§3) win
  readonly userConfigFile?: string; // absolute: the user's showcase Vite config (D138, root-only D145); merged before `config`
  readonly preludes?: readonly string[]; // absolute paths imported in order before the entry: document-level CSS and setup modules (C10's showcaseStylesheets, showcaseSetup; D90, D95; root-only D145)
  readonly title?: string; // <title>; default: data.repo.name
  readonly prepareData?: (data: ShowcaseData) => JsonValue; // L20: what is embedded; default: the data as given
}

export function buildSite(ctx: BuildContext, app: ViteApp): Promise<BuildResult>;
export function startDevServer(ctx: DevContext, app: ViteApp): Promise<DevSession>;
```

A module's `build` is then `return buildSite(ctx, { entry, plugins: [react()], … })`. C10's `-base-react` fills `ViteApp` from its options. The fields are generic, so a Vue or Svelte module could use them unchanged.

**`prepareData` (L20):**

- `-base-vite` embeds whatever `prepareData` returns. The default, `ShowcaseData` as is, is correct but carries `PackageInfo.raw` and `Doc.source`; `-base-react` (C10) passes a function that drops what its app doesn't read.
- **Contract (S12, T10):** the result keeps the hierarchy shape that `applyTestUpdate` walks: the packages keep `children`; every hierarchy node (group or unit) keeps `type` and `children`; units also keep `dir` and `tests`.
- Showcase loaders (§4) are built from the unprepared data, so `prepareData` doesn't affect them.

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
| `optimizeDeps.entries` | `app.entry`, the preludes, every showcase file (absolute, glob-escaped: entries are glob patterns, T8) | S4: the dependency scan reads entry HTML from disk, bypassing our `load` hook; the default would crawl the repo's own `index.html` (or find no entries and force a reload on first visit). Fixed per `dev` session: a showcase added later has its dependencies discovered lazily (one reload) |
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
- **Abort:** `ctx.signal` is checked before and after the Vite call. A running Vite build can't be aborted, so an abort during a build resolves after the build finishes.
- **Windows (S14):** every generated import specifier and virtual id passes through Vite's `normalizePath`.

## 4. Virtual modules (the `virtual` plugin)

The ids resolve to `\0`-prefixed virtual ids; their types live in `./client`.

| Id | Exports | Notes |
| -- | ------- | ----- |
| `virtual:tsshowcase/data` | `data` (the `prepareData` result), `dataRevision`, `testsRevision` | D77: compiled into the bundle; in `dev`, the current snapshot (§8) |
| `virtual:tsshowcase/themes` | `themes` (`ctx.themes`), `defaultTheme`, `shellCss`, `fontsCss` | both CSS strings come from our CSS pipeline (§4.1) |
| `virtual:tsshowcase/showcases` | `showcases: Record<RepoPath, () => Promise<unknown>>` | D140: one lazy `import('/abs/path')` per `ShowcaseEntry.file`, so each showcase is its own chunk (spike) |
| `virtual:tsshowcase/entry` | nothing | the HTML's only script: imports the preludes in order, then `app.entry`. It is reached as `/@tsshowcase/entry.js`, which our plugin resolves to the virtual id, the only form that works in both build and dev (spike) |

**Path leaks:** absolute paths appear only in import specifiers, which Rolldown resolves away. Debug comments are off (§3.2). The CLI's no-absolute-paths check (C11) catches anything else.

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
export function getData<T = ShowcaseData>(): T; // the embedded (prepared) data
export function onData<T = ShowcaseData>(listener: (data: T, revision: number) => void): Unsubscribe; // dev only; a no-op subscription in builds
export function onTestUpdate(listener: (update: TestResultsUpdate) => void): Unsubscribe; // dev only
export function applyTestUpdate<T extends TestTree>(data: T, update: TestResultsUpdate): T; // D141: a pure helper, typed structurally over the hierarchy (§2 contract, S12, T10); also used server-side (§8)

// showcases/
export function loadShowcase(file: RepoPath): Promise<unknown>; // D140; rejects with a clear error for an unknown file

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

**`adoptShellStylesheets(root, moduleCss)`** sets the root's sheets to `[themeShell, ...module]`. If constructable stylesheets are unavailable (feature-detected), it uses `<style>` elements in the same order instead, never mixing the two (rule 3). Every browser that can render the theme CSS (nesting: Safari 17.2, Chrome 120, Firefox 117) supports constructable stylesheets, so the fallback exists for test environments such as jsdom (our runtime tests, C10's smoke render); it is kept because it costs next to nothing (T18).

- **One shared theme sheet:** a single `CSSStyleSheet` holds the theme shell CSS and is shared by every shadow root, so one `replaceSync()` updates all of them. With the `<style>` fallback, each root gets its own element, and all of them are tracked.
- **A repeated call for the same root replaces the module sheets and keeps the theme sheet.** Modules call it again from their HMR handler when their CSS changes (§4.1).

**The fonts stylesheet** is a `<style>` element in `<head>`, not an adopted document sheet. Libraries that assign `document.adoptedStyleSheets = […]` would otherwise drop it (S14).

**Theme stylesheet updates in `dev`:**

- The runtime's themes module accepts HMR updates of `virtual:tsshowcase/themes` (§8).
- On an update, it calls `replaceSync()` on the shared theme sheet (or updates every tracked `<style>`) and replaces the fonts `<style>` text.
- A module calls `adoptShellStylesheets` once and gets these updates for free.

**Showcase loaders in `dev` (S5):**

- The runtime's showcases module accepts HMR updates of `virtual:tsshowcase/showcases`. In the accept callback: if the set of showcase files changed, it swaps its loader table; otherwise it calls **`location.reload()`** (T1).
- The second case is an edit to a showcase file that isn't a framework refresh boundary (e.g. a React showcase with a `shell` export next to its default) or to a plain helper, which propagates up to the table. `import.meta.hot.invalidate()` can't be used: the server ignores it unless the module accepts itself, and it propagates to importers rather than reloading.
- Edits handled by framework HMR (React refresh, C10) never reach it.

**`syncDocument` is the module's duty (S6):**

- `setTheme` only persists the choice, sets `<html>` `color-scheme` and notifies subscribers.
- The module renders `data-theme` from `subscribeTheme`/`getTheme` (C10, H28), and calls `syncDocument(shell)` **after** the new `data-theme` is applied (in React, a layout effect), because the computed `--tss-surface` only reflects the new theme then.
- The runtime never calls it on its own.

**`THEME_STORAGE_KEY` is shared per origin (L19).** Every GitHub Pages project site of one user shares the stored choice. That is harmless, because a stored name that isn't among the included themes falls through (rule 4). It is documented in the runtime README and in C13.

## 7. `buildSite`

1. Check `ctx.signal`.
2. **Env guard (D94, spike):** set `process.env.NODE_ENV = 'production'` for the whole of config assembly (including loading the showcase Vite config, T4) and the `vite.build()` call, and restore it in `finally`.
   - When the variable was unset before, restoring means **deleting** it, because Vite leaves it set.
   - The guard is needed because Vite derives `isProduction` and `import.meta.env.PROD` from `NODE_ENV`, not from `mode`.
   - It is safe in `build`, because no watcher runs in that process (C7 §7, C11).
3. Call `vite.build(config)` with `write: false`. The result is a single `{ output }`, since one output is protected (§3.2).
4. **Map the output to `BuildFile[]`:**
   - chunk → `{ path: fileName, contents: code }`;
   - asset → `{ path: fileName, contents: source }`, where `source` is a string or `Uint8Array`;
   - POSIX paths, sorted.
5. **Output checks** (D146; the CLI checks too, C11): a single ES output, `index.html` at the root, no file under `tss-assets/` (rule 1, `DOC_ASSETS_DIR`), and no absolute or machine-relative paths (`../` above the repo root) in any text file. A violation throws, naming the file.
6. Return `{ files }`.

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
  - `tss-assets/` (D125) serves `ctx.docAssetsDir` with `sirv`: `dev: true` (no file list cached at start, S14), GET/HEAD only, no directory listing, no dotfiles. Every other request under `tss-assets/` gets 404 and never falls through to Vite.
- **`server.fs.allow`:** Vite's default, with no additions. Module files outside the root, the theme stylesheets and their assets all load through the module graph and our CSS pipeline (spike).
- **Snapshot store and live channel (S9):** `-base-vite` keeps the current **unprepared** `ShowcaseData` and two revision counters, one per stream: `dataRevision` from `onDataUpdate`, `testsRevision` from `onTestUpdate`. The two streams come from different producers (the CLI, the module).
  - **`ctx.onDataUpdate`:**
    1. replace the snapshot and store its revision;
    2. send `tsshowcase:data` `{ revision, data: prepareData(snapshot) }` over `server.ws.send`;
    3. invalidate `virtual:tsshowcase/data` **without** an HMR update, so the next page load gets the fresh data;
    4. if the set of `ShowcaseEntry.file`s changed, reload `virtual:tsshowcase/showcases` **with** an HMR update (§6).
  - **`ctx.onTestUpdate`:**
    1. apply it to the snapshot server-side with `applyTestUpdate`;
    2. store its revision;
    3. send `tsshowcase:tests` with the update;
    4. invalidate the data module (no HMR).

    A page reload then shows current test results.
  - **`ctx.onThemeStylesheetsUpdate`:** `server.environments.client.reloadModule` on `virtual:tsshowcase/themes`. The plugin re-reads the files the CLI rewrote, and the runtime applies the result (§6). The pushed contents aren't used, because the files are the source of truth (C2 B8).
  - **Hello per subscription:** a page could miss updates sent between loading the data module and the app subscribing (an async mount). So the runtime sends `tsshowcase:hello` `{ stream, revision }` when the **first listener subscribes** to a stream (`onData`, `onTestUpdate`) and again after a reconnect, with the revision the page is showing (the embedded one at first). The server replies with the current data, or a full test state, if it has a newer revision. The runtime keeps no data copy (D141, T5); a hello sent before the socket opens is buffered by Vite's client.
  - **Stale messages:** the runtime drops any message whose revision isn't newer than the last one it applied **for that stream** (D68).
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
- **Data:** the embedded data.
- **CSS:**
  - theme CSS with a font, a large image and a small image (emitted, inlined, relative URLs), plus module CSS via `?tss-css` (not claimed by Vite's CSS plugins, T3), both **without** the PostCSS marker, while the showcase CSS has it;
  - `?#iefix` suffixes and `image-set()` string URLs;
  - `@import` in module CSS is rejected;
  - a missing `url()` target is an error;
  - `sourceMappingURL` comments are stripped;
  - `url(` in comments and selectors is untouched.
- **Chunks:** one chunk per showcase.
- **Paths:** no absolute paths and no machine-relative paths (`../`… outside the repo) in any file, also with `minify: false`.
- **Doc assets:** no files under `tss-assets/`.
- **Environment:** `NODE_ENV` is restored afterwards (also when it was unset, with a `userConfigFile`, T4, and after a failing build).

**D94:** with `NODE_ENV=development` set before `buildSite`, the result is still a production build (`import.meta.env.PROD`, the `NODE_ENV` replacement).

**`startDevServer`:**

- **Pages:** serves `index.html`; unknown paths get 404.
- **Dependency scan:** a readable repo `index.html` that imports a missing module doesn't disturb startup (S4); a repo path with glob characters still scans (T8).
- **Module CSS** via `?tss-css` is served as JS and hot-updates its importer (T3).
- **`tss-assets/`:** serves only files from `docAssetsDir`, including files added after start; traversal attempts and misses get 404.
- **Server:** `strictPort`; the non-loopback warning, also for a host set by the showcase Vite config (T6).
- **Live channel:**
  - events reach a `ws` test client in order, with per-stream revisions;
  - a test update followed by a reload of the data module shows the new results;
  - a `hello` on first subscription gets the newer state (T5);
  - a theme stylesheet update sends an HMR update for the themes module;
  - the showcases module is reloaded only when the file set changes; an edit to a showcase that isn't a refresh boundary reloads the page (T1).
- **Environment:** `-base-vite` sets nothing in `process.env`; a warning appears under `NODE_ENV=production`.
- **Shutdown:** `close()` releases the port.

### Runtime (Vitest, jsdom, S13)

- `selectTheme` against the serialised inline script (`new Function`), in every case;
- `setTheme`, `subscribeTheme` and storage failures;
- `applyTestUpdate`;
- the `<style>` fallback: order, re-adoption replacing module sheets only, and updates;
- the showcases module's accept rule (swap or `location.reload()`).

Constructable stylesheets in a real browser were covered by the spikes; as acceptance, they are covered by C12.

### README samples

Type-checked (D56).

## 10. Documentation

**Package README:**

- what `-base-vite` does;
- a minimal non-React module using it (a type-checked sample);
- the `ViteApp` fields and the `prepareData` contract;
- the protected and defaulted keys, and why;
- the showcase Vite config as a module author passes it on: root-only (D145), runs under Vite 8, plugin de-duplication;
- `?tss-css` and its rules: plain CSS, no `@import`, not minified.

**Runtime README:**

- the API;
- the order of calls: `installFontsStylesheet` → `adoptShellStylesheets` → render `data-theme` from `getTheme` → `syncDocument` after each applied theme;
- HMR of module CSS;
- the showcases reload rule;
- the per-origin storage note.

## Acceptance criteria

1. A stub module (test-only) built with `buildSite` produces a site that loads from a sub-path, embeds the data and adopts the theme shell sheet. Here this is covered by the Node-side assertions of §9; the headless check is C12's.
2. Changing any protected key of §3.2 through a user config or a plugin hook fails with an error naming the key, verified by a test per key.
3. Theme and module CSS never pass through the repo's PostCSS; showcase CSS does.
4. No absolute or machine-relative paths appear in the output, minified or not.
5. `startDevServer`:
   - serves the app;
   - keeps a snapshot that survives reloads, including test results;
   - delivers both live streams with revisions and the hello;
   - sends theme HMR updates;
   - serves `tss-assets/` from `docAssetsDir` only;
   - never sets `process.env`;
   - stops cleanly.
6. `selectTheme` and the inline script agree in every case.
7. There is no React, or any other framework, anywhere in the package (enforced by a `no-restricted-imports` rule).

## Recorded for later chunks

- **C6 (amendment, applied):** see "Amendments" below (D145, D148).
- **C8:** `-base-react`'s options schema declares `viteConfig` (D138) as a path option (`OPTION_FORMAT.path`, D116). Options are root-only by default (D148): `testRunner`, `viteConfig`, `showcaseStylesheets` and `showcaseSetup` stay unmarked (their descriptions say "root config only"), and only genuinely per-package options (e.g. analysis or rule presets) get `"x-tsshowcase-per-package": true`.
- **C10:**
  - **`ViteApp`:** fill it with plugin-react, React dedupe and resolution, a `prepareData` that drops unread fields but keeps the test tree shape, `preludes` from `showcaseStylesheets`/`showcaseSetup`, and `userConfigFile` from `viteConfig`; verify the app works when the user's own plugin-react wins de-duplication (D147);
  - **CSS:** import the module's CSS as `?tss-css` (plain CSS, no `@import`; the `-lint` module preset applies), and re-adopt module CSS on HMR;
  - **Runtime calls:** create the host and call the runtime in the documented order; call `syncDocument` in a layout effect after `data-theme` changes (S6);
  - **Showcases:** load them through `loadShowcase`. A D96 showcase with a `shell` export isn't a refresh boundary, so edits to it reload the page (S5).
- **C11:**
  - passes `cacheDir` per module;
  - rebuilds `docAssetsDir` before each `onDataUpdate`;
  - rewrites the theme stylesheet files before calling `onThemeStylesheetsUpdate` (the files are what `-base-vite` reads);
  - changing which themes are configured, or the showcase Vite config (or a file it imports), needs a restart of `dev`;
  - every `onDataUpdate` snapshot includes all test updates already forwarded through `onTestUpdate`, so the dev server's snapshot never regresses (T12).
- **C12:** in a real browser (Chromium locally):
  - constructable-sheet adoption;
  - theme asset URLs from a sub-path;
  - theme HMR, the live channel and the hello;
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
  - `dev` under `NODE_ENV=production` (D143).
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
