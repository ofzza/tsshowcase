# Super-plan — tsshowcase v1

> Status: **APPROVED** (after a final consistency review, see `01a-super-plan-review.md`); revision R1 **re-approved** together with the C2 plan (2026-10-18); revision R2 **re-approved** together with the C3 plan (2026-10-18); revision R3 (checkpoint review and prototype spikes) **re-approved** (2026-10-18); revision R4 (review of all plans before wave P3) **re-approved** (2026-10-18); revision R5 (C4 planning) **re-approved** together with the C4 plan (2026-10-18); revision R6 (C6 planning) **re-approved** together with the C6 plan (2026-10-19); revision R7 (C5 planning) **re-approved** together with the C5 plan (2026-10-19); R8 (C7 planning) **re-approved** together with the C7 plan (2026-10-19); R9 (C15 planning) **re-approved** together with the C15 plan (2026-10-19); revision R10 (the representation redesign, plan 10 of its application; review `01h-super-plan-r10-review.md`) **re-approved** (2026-10-21). Decisions referenced as `D<n>` are in AGENTS.md §3.

## Revisions

- **R1 (during C2 planning, from the C2 review `02a-c2-plan-review.md`, F37–F40):** text brought in line with decisions made after the super-plan's approval, plus chunk duties nobody owned.
  - C1: Node floor `>=22.12` (D49), CI matrix `[22, 24]` from C1 (D54).
  - C2: `showcases[]` (D61) *(R10: per (file, export), D195)*; only `analyze` required (D63); context-object signatures; module options as JSON Schema (D58); repo-level verification is `analyze` returning issues *(R10: per-file enrichments, D170)*; `build` returns files (D64); config moved out (D66).
  - C5: `Doc.headings` *(R10: `RenderedDoc.headings`, D187)*. C6: per-package config files (D59) *(R10: keys changed by D192)*. C11: the D64 write duties. C14: CI adds Node 26 (D54).
  - Added after the C2 group B decisions: `resolveRoute(target, ctx, heading?)` and routes in the data (D67) *(R10: `RouteTarget`, D183)*; `reportTestUpdate` (C10) *(R10: removed: the CLI runs tests, D172)*; `init <dir>` (C9); plain-text docs and link mapping in C5 (D71, D73) *(R10: mapping replaced by D183)*; merged per-package options in C6 (D69) *(R10: marked options only, D148)*.
  - After the remaining C2 decisions: data embedding is the module's business (D77, C2 bullet updated); `check` is a prerequisite of `build`/`dev` (D76, C11) *(R10: replaced by D164)*; `loadTestRunner` candidates (D74, C6) *(R10: replaced by D172)*; module-defined CLI flags (D79, C6/C9); group READMEs (D78, C11) *(R10: no groups, D183)*; versioning from `0.1.0` (D80: C1 amendment A2, C14).
- **R2 (during C3 planning, D81–D89, plus the C3 review `03a-c3-plan-review.md`):**
  - Packages: new published `@ofzza/tsshowcase-lint` (D88); the catalogue reference docs move from `-theme-default` to `-api` (D83); `-theme-default` gains `css/` and named file exports; the CLI gains the theme scoping transform.
  - **Isolation model (D89):** the shell lives in a shadow root (theme and module CSS adopted there), showcases are light-DOM children slotted into their stage, receive a colour scheme through a module-defined interface, and each gets a light/dark selector; regions, `@scope` and `tss-plain` are gone (D87 superseded).
  - C3: no `@layer` (D81); the CLI scopes theme CSS into a shell and a fonts stylesheet (D82, transform implemented in C3); themes may style anything inside the shell within three rules (D86) *(four since D92/D126)*; states via ARIA/data attributes (D84); contrast is a review guideline (D85); C3 adds the `-lint` package.
  - C5: catalogue classes and `tssData`. C10: the shadow-root shell, slotted showcases, scheme selectors, the shell opt-in for D41 showcases *(R10: via `ShowcasingComponent.meta`, D195)*, `-lint` presets. C11: the CLI cache, atomic writes, calls the transform, passes `themeStylesheets` (new edge C11 → C3). C13: `-lint` and the theme recommendation. C14: publishes `-lint` after `-api`. C15: adopts the shell stylesheet into the shadow root, adds the fonts stylesheet, `fs.allow` for the cache and theme assets, hot-reloads adopted stylesheets.
  - Stale text fixed: C2 section (`themeStylesheets`, inclusion, changelog from `0.1.0`), C13 changelog, C1's package list.
  - After the second C3 review (`03b-c3-plan-review.md`): C15's runtime API and C10's isolation duties made explicit (H28); C12's D89 acceptance and contrast smoke test (H33, H31); the `-base-vite`/`-base-react` rows and C10's showcases (H30).
- **R3 (checkpoint review before wave P3, `01b-checkpoint-review.md`, D91–D93):**
  - New "C2 CLI duties → chunk" checklist (K10); edge C6 → C4 and the C4 scanner/workspace-detection seam, C4 planned before C6 (K9); theme resolution split between C6 and C11 (K14).
  - C10: React fallback for repos without React (D91). C11: `build [outDir]` (D93) *(R10: `build --out`, D167)*, `check` in the data assembly (K25) *(R10: the pipeline, D168)*. C5: group targets (D78) *(R10: no groups, D183)*.
  - Stale text: C2 bullet (`onThemeStylesheetsUpdate`, `dispose`), C15 `dev()` line, C11 "Depends on", the P3 rationale, the closing note on contract gaps (K12).
  - Open points for every remaining chunk now live in AGENTS.md ("Open points per chunk"). A "v1 cut line" review runs once all chunk plans are approved, before implementation (K21).
  - Prototype spikes instead of gates (K17/K20, `01c-spike-results.md`): the in-process Vitest runner is proven (C7, D94) and the D89 technique is confirmed in Chromium, Firefox and WebKit, with fixes (C10, C13, C15, D95).
- **R4 (review of all plans before wave P3, `01d-pre-p3-review.md`, D96–D102):**
  - C8: a preliminary React showcase file format (D96) *(R10: changed by D195)*. C10: finalises it; theme-owned breakpoints (D97); React resolved per showcased package (refines D91).
  - L1–L3 recorded as known open issues (AGENTS §4); the rest as open points per chunk. The cut-line review stays after all plans (D98); C10 stays one chunk (D99); shell-mode showcases get no scheme selector (D100 *(R10: opt-in via `meta`, D195)*; C2 rule 7, C3 §1); `include` folders and the marker-file skip (D101 *(R10: `scope`, package-relative, D165)*; C2 §5.5, §10, C4, C6); `BuildContext.themes` carries only name, label and scheme (D102, C2 §5.8).

- **R5 (C4 planning, D103–D110, reviews `04a-c4-plan-review.md`):**
  - **What exists (D103, D109):** hard ignores, then config `ignore` (renamed from `exclude`), then `.gitignore` and the `.tsshowcaseignore` alias, at or below the root only; git's parent rule for every source; no `.git/info/exclude`; no git calls.
  - **Renames (D107):** `include` → `scope` (where to look, never un-ignores), `exclude` → `ignore`; scanner inputs `{ scope, ignore, skippedPackages, outDir }` *(R10: `scope` per package, D165)*.
  - **Access (D108):** `RepoFiles`/`DirectoryInput` *(R10: `DirectoryInput` removed, D177)* treat ignored paths as nonexistent unless `{ ignored: true }`; `.git`/`node_modules` never reachable (C2 amendment B4).
  - **C8 → C9 (D106):** units (and `analyze` issues) outside `scope` are dropped by the CLI, not the module *(R10: replaced by in-scope results, D165, D170)*.
  - **VCS (D110):** from `package.json#repository` and config `vcs.*` only, links always use `HEAD`; no git (revises D24, D105) *(R10: links point at `vcs.ref`, default the commit read from `.git`, D205)*.
  - **New private package `-test` (D104)** for test helpers (C1 amendment A3).

- **R6 (C6 planning, D111–D117, review `06a-c6-plan-review.md`):**
  - C6: plugin paths only as `file:` (D114); config files `.ts|.mts|.js|.mjs|.json|.jsonc`, JSON with comments (D115); C6 parses every option flag and positional, C9 owns the commands and `--help` (D111); flags vs config arrays (D112); `dev.host`/`dev.port` (D113); path-marked module options (D116, C2 amendment B5); path-loaded plugins get the CLI's tsshowcase packages (D117); `outDir` resolved by C6 *(R10: `build --out`, D167)*.
  - C9: "CLI framework, global flags" → the command framework and `--help` (D111). C11: `--out` → `build [outDir]` (D93, stale since R3) *(R10: back to `build --out`, D167)*.

- **R7 (C5 planning, D118–D131, review `05a-c5-plan-review.md`):**
  - C5: starry-night (D118); sanitised raw HTML (D119); local README media copied into the site as doc assets by the CLI, served from a CLI-maintained directory in `dev` (L1, D120, D125, D129) *(R10: only the assets of rendered docs are written, D187)*; broken README links and anchors as `check` warnings `docs/broken-link`, `docs/broken-anchor`, decided without routes (D123, D124); alerts (D122); exact case (D130) *(R10: extension-based classification is case-insensitive, D201)*. C5 now depends on C4 (the file layer).
  - The hierarchy helper (groups, nesting, group READMEs) moves from C11 to C9 (shared, reused by C11) *(R10: replaced by the representation builder, D179)*; C9 reserves a docs step in `check`, wired by C5's implementation; C9 depends on C5. Implementation order unchanged *(R10: still true; the order is now D194's, with C7 before C9)*.
  - C11 writes doc assets and maintains the `dev` doc-assets directory; C15 serves it; C12 gains README image, anchor and alert checks. C2 amendment B6, C3 amendment C3-A1, a C6 amendment (`docs/*` rule ids) and a C4 note.

- **R8 (C7 planning, D132–D137, review and spikes `07a-c7-plan-review.md`):**
  - C7: the runner decides where Vitest runs (root or per package, configs looked up only in each run directory, D132, D134); `detect` is "Vitest declared or a Vitest config"; `vitest/node` resolves from each run directory; the refined process rule (restore timing, removing Vitest's process handlers); newer majors tried with a warning (D133); the environment is left alone (D137); tests call the runner in-process (spike).
  - C10: `runTests` passes `ctx.repo`; units under `failedRoots` are `unavailable` (D135). `testRunner` is root-only (D136, C6 warns in package configs, C8 schema). *(R10: replaced by D172: the CLI owns the runners; `testRunner` is a root-only CLI key)*
  - C2 amendment B7, a C4 amendment (`copyFixture` links every `node_modules`), a C6 amendment (`testRunner` in package configs, runners get `ctx.repo`; *R10: void since D172*).

- **R9 (C15 planning, D138–D145, spike `15a-c15-spike-results.md`, review `15b-c15-plan-review.md`):**
  - C15 rewritten from its plan: own CSS pipeline for theme and module CSS (`?tss-css`) instead of `?inline` processing, no `fs.allow` additions (D144); the showcase Vite config option (D138), `.env` off (D139), showcase loaders (D140) *(R10: per file with showcasing components, D195)*, test updates (D141) *(R10: over `Representation.tests`, D186)*, `resolve.tsconfigPaths` (D142), `dev` never sets `process.env` (D143); protected config keys *(R10: asserted, not re-applied, D146)*; a dev snapshot that survives reloads.
  - C10: `syncDocument` after `data-theme` is applied, module CSS as `?tss-css`, the `viteConfig` option (root-only with `showcaseStylesheets`/`showcaseSetup`, D145). C11: a change to the showcase Vite config needs a `dev` restart.
  - C2 amendment B8 and C3 amendment C3-A2 (wording); a C6 amendment (module options root-only by default, per-package ones marked in the schema, D145, D148).

- **R10 (the representation redesign, D160–D210; plan 10 of its application, `08b-representation-redesign.md` §12, `08e` E7; review in `01h-super-plan-r10-review.md`):** units, groups and module-owned checks and tests are replaced by the representation model: the CLI scans, associates resources with public symbols, builds a representation (nodes), runs the check and the test runners; the module enriches (symbols, kinds, showcasing components), verifies per file and presents (D160, D166, D168). R1–R9 above are history, their superseded items annotated *(R10: …)*; where they differ from the text below, the text below wins.
  - **Packages:** `-api` (the `Representation` data model), the CLI (associations, the representation, the check, directory configs, the test runners), `-testrunner-vitest` (the CLI's runner: lists and runs), `-base-vite` (embeds the `Representation`, rendered docs as lazy chunks, showcases per (file, export)), `-base-react` (per-file analysis and `verify`, no `runTests`) and `-module-react-default` (nodes, homes, `homeDocs`).
  - **Chunk texts:** C1 (the dogfood layout of C1 A4, the template import of A5, D203); C2 (the B9/B10 contract); C4 (per-package in-scope trees, homes with roles, directory-config files, `vcs.ref`, D205); C5 (lazy `RenderedDoc`s, D183 link resolution, doc-id prefixes, `data-link` states); C6 (runner loading, directory configs after the scan, the argv normaliser, `--out`, `--strict`, `--rule`, the rule-id table, D206–D208); C7 (`listTestFiles`, the CLI intersects, D209); C8 (a stub until its rewrite in plan 11); C9 (the pipeline's steps 1–7, `init`); C10 (rendering the `Representation`); C11 (the pipeline's step 8, CLI-run tests, `--strict` parts, `--no-tests`, lazy docs); C12–C15 accordingly (C15: one revision, `selectDocs`, doc chunks, `loadDoc`, D210; protected keys asserted, not re-applied, D146).
  - **The C2 CLI-duties table rewritten** (scan, directory configs, runners, analysis validation, associations, representation, check, issue placement, overrides, `--strict`, tests, lazy docs, assets, `dev` invalidation).
  - **Edges:** C9 → C7 added and C10 → C7 dropped (D194; C11 keeps C7); C7 → C4, C6 added (`RepoFiles`, `copyFixture`, the CLI's runner loading in its tests, `07b`). **Implementation order:** C7 before C9 (D194).
  - Stale text fixed: the issue model's `file` → `path` (D202); "links always use `HEAD`; no git" → `vcs.ref` (D205); the hierarchy helper, group READMEs, `runTests`, `loadTestRunner`, `--fail-on-test-failure`, `--no-check`, `build [outDir]` and the per-stream revisions removed.

## Packages (D14, D15, D26, D27, D43)

| Package                                  | Role |
| ---------------------------------------- | ---- |
| `@ofzza/tsshowcase-api`                  | Contracts only: TS types and interfaces (the `Representation` data model, the module, test-runner and theme interfaces, the issue model), `IMPLEMENTED_API_VERSION`, interface changelog, the theme catalogue constants, small runtime helpers (e.g. issue constructors). Has no heavy deps. |
| `@ofzza/tsshowcase`                      | The CLI (binary `tsshowcase`): repo scanning, directory configs, the markdown pipeline, plugin loading (modules, themes, test runners), config, associations, the representation, the check, running tests, theme scoping (PostCSS, D82), `dev` file watching, and the `check` / `build` / `dev` / `init` actions (D168). It doesn't bundle anything itself; modules build (D43). Depends on the built-in packages so they can be looked up by name (D9). |
| `@ofzza/tsshowcase-theme-default`        | Default themes: built-in names `default-light` and `default-dark`; CSS in `css/` (`files: ["dist", "css"]`), named file exports for composing themes, user-facing theming docs. The catalogue reference docs live in `-api` (D83). |
| `@ofzza/tsshowcase-lint`                 | Published lint rules, plugins and presets for tsshowcase plugins (D88): a stylelint plugin and an ESLint plugin enforcing D46/D86, presets per plugin kind (theme CSS, module CSS, module TSX). Enforced on this repo's own theme and module packages; opt-in for third-party authors. Contains no code that runs linting. |
| `@ofzza/tsshowcase-testrunner-vitest`    | The CLI's built-in Vitest test runner (D172), with listing, run and watch modes. Built-in name: `vitest`. |
| `@ofzza/tsshowcase-base-vite`            | Reusable Vite pipeline helper (D43). Not a loadable module. Handles: embedding the module's prepared data (`prepareData` over the `Representation`, keeping `revision` and `tests`) as a virtual module, the docs the app selects rendered and embedded as lazy chunks (D210), showcase loaders per (file, export) (D195), a framework-agnostic runtime API for data, docs, showcases, theme stylesheets (adopted into a given shadow root, fonts stylesheet in the document, dev updates) and theme state (D89), `base: './'`, in-memory build output (returned as files for the CLI to write, D64), dev server, HMR and the live update channel. Modules call it with their framework plugins and app entry. |
| `@ofzza/tsshowcase-base-react`           | Shared React base (D20, D26, D48). Layout-agnostic and **not itself a loadable module**. Two halves:<br>• **check/init (Node, no bundler):** static entry analysis returning per-file enrichments (symbols with kinds, showcasing components, D170, D195), per-file `verify` rules incl. the `headless` preset (D159, D177), scaffold templates (D196);<br>• **build/dev:** a **React wrapper around `-base-vite`** (plugin-react, React deduped from the showcased repo, tsconfig paths) plus the runtime building blocks (data and live-channel access, the shadow-root shell host, slotted showcases with per-showcase scheme selectors, rendered-doc container, test-results and issues views, theme switcher, hash router with in-shadow anchor scroll; D89).<br>Exposes a **base module class** with analyze / verify / resolveRoute / scaffold / build / dev already implemented. Layout modules extend it and supply only their app entry and layout. Depends on `-base-vite`. |
| `@ofzza/tsshowcase-test`                 | **Private, never published** (D104): test helpers for this repo's own tests (`copyFixture()`, `fixturePath()`); a devDependency only; skips itself from the dogfood check (`skip: true`). Added by C4 (C1 amendment A3). |
| `@ofzza/tsshowcase-module-react-default` | The v1 rendering module (D22). Built-in name: `react-default`. Contains **layout only**: app shell, navigation tree (homes → packages → nodes), route scheme, and the home, package and node pages composed from the base building blocks; which home docs are shown (`homeDocs`, D185). |

## Chunks

Each chunk lists **Goal**, **Requirements** and **Depends on**. Each chunk delivers its own docs, its tests and the fixtures it needs (D10 conventions; docs ship with code).

### C1 — Monorepo foundation

- **Goal:** an empty but fully wired workspace.
- **Requirements:**
  - **The TS template is the source (D52, D203):** at the start of the implementation, what suits is copied from `~/projects/ofzza/.templates/ts` by hand and merged interactively with the user; the import is logged in `TEMPLATE.md` (template commit, per file: as is / merged or changed / left out, with reasons).
  - npm workspaces with the package skeletons listed above (except `-lint`, added by C3, D88, and the private `-test`, added by C4, D104).
  - TS (ESM, Node `>=22.12`, D17/D49), build via `tsc` project references.
  - Prettier (160), ESLint (incl. no `export *` in entry files, C1 A4), Vitest workspace config (one project per package, extra `unit:<pkg>:<part>` projects where needed, type tests incl. `*.test-d.{ts,tsx}`), `npm run ci` aggregate script.
  - A `fixtures/` directory convention. Fixtures are **isolated from the workspace**: excluded from `workspaces`, from root lint/test/type-check, and from dependency hoisting (each fixture has its own install where it needs `react`/`vitest`). Otherwise "resolve React from the showcased repo" can't be tested. `fixtures/` lies outside every package, so it is never in scope (D165).
  - **Basic CI (D32):** a GitHub Actions workflow that runs lint, type-check, test and build on push/PR on a Node `[22, 24]` matrix (D54), so every chunk is checked.
  - changesets set up for independent versioning (D31).
  - The D10 conventions documented in AGENTS.md.
  - **Dogfooding (D41, C1 A4):** the package layout follows the representation rules: one exporting file per concept (`src/<area>/<Concept>.ts(x)`), a README per directory with exporting files documenting every exporting file in it, tests and showcases named after the file they cover (as the kinds require, D162), entry files re-exporting declaring files by name, kinds by value (D200); this repo's root `tsshowcase.config.json` (`scope: ['src']`, `strict: ['check', 'warnings']`) is written in C9.
- **Depends on:** nothing.

### C2 — Plugin contracts (`-api`) ★ critical path, designed collaboratively in detail before any code (assignment requirement)

- **Goal:** the complete, documented v1 contracts.
- **Requirements:**
  - **Input data model:**
    - the scan layer: `RepoInput` (repo info, VCS info with `ref`, D205; packages with info, `scope` and options), `RepoFiles`, the in-scope `ScannedTree` with file roles (D169);
    - packages[] (D16), package info sourced from `package.json` in v1 but source-agnostic so other repo types can come later;
    - **`Representation` (D187):** homes with role-tagged docs (D185; single-package repos share one home), package roots and a tree of `RepresentableNode`s (D178) with their items (symbols with kinds, D162, D200), associated docs (`DocRef`), tests and showcases per (file, export) (D195), each with an association level (D173), issues (D180), routes (D183) and `meta` passthrough; tests in the data with one shared revision (D186); `TestsUpdate`;
    - extensible: additive only (D13).
  - **Rendering module interface** (default-exported class, D9 loading). Only `analyze` is required; every other method is optional and feature-detected per action (D198). Methods take context objects:
    - statics `apiVersion` (semver range, D13), `displayName`, `rules` (the module's own rule ids, outside the CLI's reserved namespaces, D175), `kinds` (module kinds and their requirements, D162), `testRunners` (D172), `optionFlags` (D79), `optionsSchema`;
    - `analyze(ctx)` → per-file enrichments (symbols, showcasing components, `meta`), issues, `inputs`, `state` (D170, D171);
    - `verify(file, ctx)` per in-scope file → issues (D177);
    - `resolveRoute(target, ctx)` (D183): the in-app URL for a node, an optional doc and anchor or fragment (D204), or `undefined` (the VCS host); the CLI stores routes in the data;
    - `build(ctx)` and `dev(ctx)` (D43): the module builds the static app and returns it as files for the CLI to write (D64), or runs a dev server. `ctx` carries the `Representation`, lazy `docs` (D187), the themes (names, labels, schemes, D102), the CLI-scoped `themeStylesheets` (D82) and (in dev) `onDataUpdate` / `onTestUpdate` / `onThemeStylesheetsUpdate` subscriptions;
    - `scaffold(ctx)` for `init` (D196): returns files for the CLI to write (D64);
    - `dispose()` at the end of every action;
    - module-specific options schema, as JSON Schema (D6, D58), with path and reference options marked (`OPTION_FORMAT`, D116) and per-package options marked (D148).
  - **Inclusion rules for any module's output** (D43): how the CLI-scoped theme stylesheets are included (D82), how the switcher attribute works, showcase isolation (D89), prefixed element ids (D184), static/relocatable output with no leaks. Mandatory even for modules that don't use `-base-vite`. How the data is embedded is the module's business (D77).
  - **Issue model:** severity, message, `path` (a file or directory, D202), `line`, rule id; rule namespaces and the CLI's rule-id table (D175); severity overrides with prefix patterns (D176, D206); `--strict` parts (D164).
  - **Test-runner interface:** `apiVersion`, `detect(repo)`, `listTestFiles(options)` (D172), `run(options)`, `watch(options, onUpdate)`; a raw result hierarchy (file → suite → test, with status, duration and error); never throws on failing tests; the CLI never runs a runner alongside another runner, `analyze`, directory-config loading or `vite.build` (`07b` QC).
  - **Theme contract:** a theme manifest (`apiVersion`, name(s), CSS entry files, light/dark flags, D7).
  - **Theme catalogue constants (D28):** every documented CSS class name and CSS variable name, exported as typed constants that modules import. The _values_ and the _design_ of the hierarchy come from C3; C2 provides the slot and the export mechanism, and C3's plan fills it in (it extends `-api`).
  - **Data handed to `build`/`dev`:** the `Representation`, `DocsAccess` and the dev update protocol (`DataUpdate`, `TestsUpdate`, one revision, D186).
  - **Output tolerance:** the CLI tolerates unknown fields in module outputs (analysis results, issues, test results) as well as in inputs (D13).
  - Interface changelog: `-api`'s changesets `CHANGELOG.md`, starting at `0.1.0` (D80).
- **Depends on:** C1 (only for where the code lives; planning is independent).
- **Note:** C3 fills in the catalogue constants in `-api` (D28). Implementing C3 modifies the `-api` package.

### C3 — Theme system + default themes

- **Goal:** design the CSS variable hierarchy and class catalogue (the theme ⇄ module contract; published as constants in `-api`, D28), plus the light and dark themes.
- **Requirements:**
  - **Layered variables:**
    - base tokens (primary/secondary colours, base spacing, base font size, radius, …);
    - → derived scales (palette ramps, spacing scale, type scale) computed from the base tokens (`calc()`, `color-mix()`);
    - → semantic tokens (surface, text, link, border, code background, success/failure, severities, …);
    - → component tokens.
  - Overriding any layer cascades to the layers below it.
  - Documented, `tss-`-prefixed class catalogue (states as ARIA/data attributes, D84): layout shell, nav, headings, paragraphs, links (incl. the `data-link` states for broken and info-less VCS links, D183), lists, tables, code (inline and block), blockquote, doc content and text docs, symbols, showcase frame and error, test results (runners, projects, errors), issues, theme switcher (C3-A3).
  - Covers the markdown HTML output (C5).
  - Runtime theme switching with a `prefers-color-scheme` default (D7). Persists the user's choice and avoids a flash of the wrong theme.
  - **Isolation model (D89):** specified here (shadow-root shell, slotted light-DOM showcases, inheritance resets, colour scheme for showcases, anchors to any prefixed element id, D184); implemented in C10/C15.
  - **Theme scoping:** how several themes coexist in one bundle (e.g. `[data-theme=…]` scoping), and how a custom theme can extend a built-in one (by composition: theme CSS is unscoped and the build scopes it, D65).
  - **Theme rules (D86):** a theme may style anything inside the shell; no self-scoping or document selectors; own names use their own prefix; modules use catalogue names. No cascade `@layer` (D81).
  - Prefix `tss-` / `--tss-` (D45).
  - **Scoping transform (D82):** CLI-side, implemented here, wired up in C11.
  - **Enforcement tooling (D46, D88):** one policy, shipped as the published `@ofzza/tsshowcase-lint` (rules, plugins, presets), enforced on tsshowcase's own module and theme packages and opt-in for third-party authors:
    - allowed: `var(--tss-…)`, `calc()`/`color-mix()` of them, structural keywords;
    - banned: literal design values;
    - no self-scoping or document selectors; own names never use the `tss-` prefix (D86);
    - escape hatch: a line-level disable comment with a required reason.
  - Docs listing every variable and class.
- **Depends on:** C2 (theme manifest shape only). The catalogue design itself is independent.

### C4 — Repository scanning (CLI core)

- **Goal:** turn a repo on disk into the scan layer of the C2 input model: `RepoInput`, `RepoFiles`, per-package in-scope trees (`ScannedTree`), homes and directory-config files. Symbols come from the module's `analyze` (C8), test files from the runners (C7, through C9), associations and the representation from C9, HTML from C5.
- **Requirements** (details: the approved C4 plan):
  - **What exists (D103, D109):** hard ignores (`.git`, `node_modules`, an explicitly configured output directory, any directory holding tsshowcase's output marker file, skipped packages); then config `ignore` patterns (gitignore syntax incl. `!`, overriding ignore files on the same path); then `.gitignore` files and their `.tsshowcaseignore` alias at or below the root only, with git's semantics (no `.git/info/exclude`, no global excludes, tracking state irrelevant, no git calls). Git's parent rule holds for every source. Behaviour is the same for non-git directories.
  - **`scope` (D165):** paths and glob patterns relative to each package (default `['.']`), matching directories; it never un-ignores; directories outside packages are never in scope; an entry matching in no package is a printed warning, not an issue. Repo metadata is read wherever it lives.
  - **The walk (D169):** one typed tree per package, stopping at nested packages, files before subdirectories; scan-time roles `doc` and `config`, the every-role tree added later (C9).
  - Lazy file content accessors; `read`/`exists`/`list` treat ignored paths as nonexistent unless `{ ignored: true }` (D108). Read tracking for `RepoFiles`, per `verify` call (D177, K10).
  - Workspace detection (npm, yarn and pnpm workspaces); single package = one package (D16). **A separate function** that C6 calls first, to find per-package config files (K9).
  - **The scanner takes its exclusions as inputs:** `{ scope (per package), ignore, skippedPackages, outDir }`, resolved by C6 from the config (K9, D107).
  - package.json info, licence detection (D12); **homes (D185):** every root markdown doc plus `README`/`LICEN[CS]E`/`CHANGELOG` text forms, with roles, in the default home order (D178); doc classification with extensions compared case-insensitively (D201).
  - **Directory-config files (D188):** found in in-scope directories only; loaded by C6, merged by C9.
  - **VCS info (D24, D110, D205):** from the root `package.json#repository`, overridden by config `vcs.*` and flags; links point at `vcs.ref` (default `CURRENT`, the commit read from `.git` by a small reader, without running git).
  - **Scan issues:** `scan/invalid-manifest`, `scan/no-manifest`, `scan/unreadable` (D175).
  - Fixtures: a single-package repo, workspace repos, and broken variants; the `copyFixture()` helper in the private `-test` package (D104).
- **Depends on:** C2.

### C5 — Markdown pipeline (CLI core)

- **Goal:** markdown → themed HTML (D23), rendered lazily per `DocRef` behind `DocsAccess` (D187).
- **Requirements:**
  - unified/remark/rehype with GFM and heading slugs.
  - Emits only C3 catalogue classes (and `tssData` attributes where needed).
  - **Link resolution (D183):** the CLI picks the node (a doc's own nodes, else its associated nodes, read from C9's `PathNodeIndex`; homes for home docs and repo/package roots), the module the URL through `resolveRoute(target)`; `undefined`, out-of-scope paths and repo files without a node → the VCS host URL template (GitHub/GitLab/Bitbucket, overridable, D24, D205); external links untouched; anchors on doc targets as `RouteTarget.anchor`, fragments on other node targets as `RouteTarget.fragment` (D204); broken and info-less VCS links as `<a>` without `href` with `data-link="broken"` / `"unlinked"`; links within the same doc marked `data-anchor` (`05b` Q3).
  - **Doc ids and element ids (D184):** every element id in a rendered doc is prefixed with the doc id (headings, footnotes, author anchors); permalinks to the link-selection node.
  - Collects each document's headings (depth, text, id) for `RenderedDoc.headings` (C2).
  - Plain-text docs (`LICENSE`, `.txt`; `DocRef.format: 'text'`, D71) render as escaped `<pre>` with a catalogue class.
  - Syntax highlighting for code blocks (D39), emitting only themable token classes, with no inline colours: starry-night (D118).
  - Raw HTML passed through, sanitised with GitHub's rules, author classes stripped (D119, D126); GitHub alerts (D122); GitHub-compatible heading and footnote slugs, author anchors (D128).
  - **Doc assets (L1, D120, D129, D187):** local doc media copied into the site under `tss-assets/<hash>/<name>` (`RenderedDoc.assets`), from the local state, only files whose real path lies inside the repo and isn't ignored; only the assets of rendered docs are written in `build` (C11, `renderedAssets()`); in `dev` the CLI pre-copies every `DocRef`'s assets into a directory the module serves (C11, C15).
  - **`check` issues (D123, D124):** `docs/broken-link`, `docs/broken-anchor` (warnings) from `checkDocs` over every `DocRef`, which shares the classifier with rendering and never needs routes; exact case on every filesystem (D130; extension-based classification is case-insensitive, D201).
- **Depends on:** C2, C3 (class names), C4 (the file layer). Uses C9's representation and `PathNodeIndex`, so it is implemented after C9 (D194).

### C6 — Plugin loading + configuration (CLI core)

- **Goal:** resolve and validate modules, themes and test runners, and the effective config. For themes, C6 resolves references (D9, `#name`), loads and version-checks theme sets; C11 selects the configured themes, checks duplicate names and `defaultTheme`, and scopes (K14).
- **Requirements:**
  - Reference resolution: built-in name → npm package; paths only as `file:<path>` (D9, D114). Path-loaded plugins and config files get the CLI's own `@ofzza/tsshowcase*` packages (D117).
  - TS loading via jiti.
  - Default-export class instantiation; static checks of module `kinds` (no collisions with the well-known kinds, key pattern), `testRunners` and declared rules outside the CLI's reserved namespaces.
  - `apiVersion` semver check with clear errors (D13).
  - **Test-runner loading (D172, D209):** candidates from the module's static `testRunners`, the built-ins and config `testRunners`, de-duplicated; config `testRunner` pins runners and skips `detect`; otherwise every detecting runner is used; `listTestFiles` required; ids are slugified display names; an unpinned candidate failing to load or a throwing `detect` → `tests/runner-failed`, a pinned runner failing to load → a thrown `PluginError`.
  - Module-defined CLI flags mapped onto option paths (`optionFlags`, D79): parsing from the schema, precedence over config, collisions, `--help`.
  - The config schemas (D6, D58, D62), owned by the CLI package, not `-api` (D66): `config.schema.json`, `package-config.schema.json`, `directory-config.schema.json`, inferred types, `defineConfig`, `definePackageConfig`, `defineDirectoryConfig`, documented in the CLI README; the redesign's keys (D188–D192), removed keys get only a generic "did you mean" (D208).
  - Config file discovery (`tsshowcase.config.ts|mts|js|mjs|json|jsonc`, JSON with comments, several files an error, D115), validated against that schema, merged with CLI flags (D6, D112).
  - **Directory configs (D188):** loaded and validated after C4's scan (`loadDirectoryConfigs`), raising `config/several-configs`; merged by C9.
  - Argv parsing: every option flag and positional in one registry, two-phase `parseArgs` (D111); the argv normaliser for `--strict`, `--no-strict` and `--rule`, applied in command-line order and only on the actions where they have an effect (D164, D176, D207); `--out` (D167), `--no-tests` (D197); path-marked module options resolved per source before merging (D116).
  - Per-package config files in workspace packages (D59, D192), for packages that exist only: discovery (using C4's workspace detection), the restricted package schema, and merging over the root config (`scope` per package, D165); only module options marked per-package are merged (D148), each package's merged `moduleOptions` validated against the module's `optionsSchema` and handed over as `PackageInput.options` (D69).
  - Validation of module options; `rules` keys validated against the CLI's rule-id table and the module's rules, with prefix patterns (D175, D176, D206).
  - **Scan and VCS keys:** `scope` and `ignore` (D107, D165) with repeatable `--scope` / `--ignore` flags, `vcs.webUrl` / `vcs.directory` / `vcs.host` / `vcs.fileUrlTemplate` / `vcs.ref` with `--vcs-*` flags (D110, D205); resolved into C4's scanner inputs.
- **Depends on:** C2, C4 (workspace detection, scanner inputs and the scan for directory configs; K9).

### C7 — Vitest test runner (`-testrunner-vitest`)

- **Goal:** the CLI's built-in runner (D172): lists and runs the showcased repo's tests; the CLI intersects the results with the in-scope tree and stores them (D186). No module mapping.
- **Requirements:**
  - Runs the showcased repo's Vitest via the Vitest Node API, using the repo's own vitest install and config. **The runner decides where it runs (R8, D132, D134):** once at the repo root, or once per package, with configs looked up only in each run directory and passed explicitly; listing and running share the plan, frozen while a watch is active; a failing per-package instance reports `failedRoots`.
  - **`listTestFiles` (D172):** through `globTestSpecifications()` under the process guard, one entry per `(file, project)`, in-source tests included; during a `watch`, fresh instances for the watch's plan.
  - Returns the **raw** C2 test hierarchy (file → describe → test, with status, duration and error message).
  - Watch mode that streams updates (D18).
  - `detect(repo)`: Vitest declared in a manifest, or a `vitest.config.*` / (3.x) workspace file (R8: not a `vite.config` alone, not "`vitest/node` resolves"); never throws.
  - A failing test never throws.
  - **Proven by the spike (`01c-spike-results.md`, D94):** `createVitest` + the reporter API (`onTestRunStart`, `onTestModuleEnd`, `onTestRunEnd`), `vitest/node` resolved via `createRequire` from each run directory (R8); supported range `^3.2 || ^4 || ^5`, newer majors tried with a warning (D133); restore `process.exitCode` and `process.env` (in `watch`: the exit code after every rerun, the env when watching stops) and remove Vitest's process signal/exit handlers (R8, C2 B7); leave the environment otherwise alone (D137); "no tests" = completed; results carry `project` and `unhandledErrors`; no port/HMR isolation needed next to our Vite.
- **Depends on:** C2, C4 (`RepoFiles`, `copyFixture`), C6 (the CLI's runner loading, used in its tests).

### C8 — React base: Node side (`-base-react`)

- **Goal:** everything `check` and `init` need for React/TS repos, plus the loadable `-module-react-default` class, so that C9 can run `--module react-default` before C10 exists. No bundler involvement (D48).
- **Requirements:** rewritten on the representation model in plan 11 of the redesign's application (the C8 plan rewrite, then this text); until then: 08b §12 ("C8") and the AGENTS open points for C8.
- **Depends on:** C2, C4 (fixtures).

### C9 — CLI shell + `check` + `init`

- **Goal:** the first usable vertical slice: the pipeline's steps 1–7 (D168).
- **Requirements:**
  - The command framework (action dispatch) and `--help` rendering from C6's `describeFlags`; global flags (`--module`, `--theme`, `--config`, `--cwd`) are parsed by C6 (D111); action flags (`--format`) and positionals (`init <dir>`) declared in C6's registry.
  - **The pipeline, steps 1–7:** scan (C4) → enrichments (directory configs loaded by C6 and merged here, D188; test listing through the runners, D172, one runner at a time, never alongside `analyze`; the module's `analyze`, validated, D170) and kind overrides (D189), exemptions (D191), collections (D190) → associations (D173) → the representation (D178, D179; doc ids, D184; `PathNodeIndex`) → the check (D174) → `verify` per in-scope file (D177) → issues placed (D180), overridden (D176) and printed. Shared with C11.
  - **`check`:**
    - tree report of every in-scope directory and its nodes, with their issues (human-readable, coloured; D165);
    - `--format json` (D60); warnings on stderr, never in the JSON report;
    - non-zero exit code on errors (D16); `--strict=warnings` (D164).
  - **A docs step in `check` (D123, D183):** reserved here, wired to C5's `checkDocs` over every `DocRef` by C5's implementation.
  - **`init <dir>` (D196):** runs steps 1–4, calls the module's `scaffold` and writes the returned files (C2 §5.10) (and reports clearly when a module lacks it, per D13 feature detection).
  - **Self-check (D41):** this repo's root `tsshowcase.config.json` (`scope: ['src']`, `strict: ['check', 'warnings']`, C1 A4); `tsshowcase check` runs on this repo in CI.
- **Depends on:** C4, C5 (the docs step's API), C6, C7 (test listing, D194), C8.

### C10 — React runtime building blocks + default module (`-base-react` runtime, `-module-react-default`)

- **Goal:** the generated SPA. The base module class gains `build`/`dev`, implemented as a React wrapper around `-base-vite` (C15), plus the runtime building blocks. `-module-react-default` adds only its app entry and layout (D48).
- **Requirements:**
  - **React input to `-base-vite` (C15)** and `build`/`dev` implementations that call it:
    - `@vitejs/plugin-react`;
    - react/react-dom **resolved and deduped from the showcased repo**; a repo without React gets a React bundled with `-base-react` (D91) (pitfall: duplicate React). The dedupe also covers module runtime files living in the CLI's `node_modules`, and the module's packages must not pull in their own React copy through peer auto-install;
    - respects the repo's tsconfig paths;
  - **App, rendering the `Representation` (D187):**
    - hash router; routes per node via `resolveRoute` (D183), incl. line-anchor fragments sent to the VCS host (D204);
    - nav tree (homes → packages → nodes);
    - homes with role-based doc selection and the `homeDocs` option (D185); node pages with their items, rendered docs (`selectDocs`, `loadDoc`, D210), showcases per (file, export) (zero or more) each in Suspense and an error boundary, `shell` via `meta` (D195), tests (weak ones collapsed by default, D181), coverage and per-node summaries as the module's own notions (D182), association levels and issues;
    - theme switcher UI component (persistence and selection live in C15).
    - showcases of the `-base-react` components themselves (dogfooding, D41), using mock data.
  - **Isolation (D89, C3 §1):** the `<tsshowcase-app>` host and its shadow root (theme shell stylesheet adopted first, then module CSS, via C15's runtime API; `:host { all: initial !important }`; host as a stacking context; document scrolling, no clipping on stage ancestors); each showcase portalled into a light-DOM `[data-tss-showcase]` wrapper assigned to a named slot in its stage (`slot { all: initial !important; display: contents !important; color-scheme }`); the `showcaseStylesheets` and `showcaseSetup` module options (document-level CSS for app typography, H1; a setup module run before showcases mount, e.g. Fluent's Tabster Shadow-DOM API, D95); `isolation: isolate !important` on the host and `!important` on every declaration after `all: initial !important`; no shell handlers reacting to synthetic events bubbled from showcases; every showcase gets a `colorScheme` prop (the React showcase interface) and its own light/dark selector (follows the theme until changed, reset on theme switch, not persisted) with `data-scheme` on the stage; a showcase can opt into rendering inside the shell (D41 showcases, D195); shell overlays inside `.tss-shell`; `data-theme` rendered from C15's theme state; the router scrolls to and focuses route-targeted element ids inside the shadow root (`aria-current="location"`, D184); links marked `data-anchor` scroll on the current page when the element is there, else follow the `href` (`05b` Q3); key events from shell inputs stopped at the host, the stage labelled from inside the shadow tree; the smoke render passes a scheme and feature-detects constructable stylesheets (`<style>` fallback).
  - **Responsive layout (D97):** breakpoints live in theme CSS, which overrides layout tokens in its own media queries; module CSS has no length-based media queries. The needed tokens and missing catalogue classes come as a C3 amendment, framed as generic additions reviewed against "any module".
  - **Only C3 classes and variables are used** (plus own-prefixed arrangement classes, D86); passes the `-lint` presets (`stylelintConfigs.module`, `eslintConfigs.module`); `tssData` for states.
  - Consumes the embedded data and the dev live channel provided by `-base-vite` (C15): fills `ViteApp` (plugin-react, React resolution, `selectDocs`, `prepareData` keeping `revision` and `tests`, preludes from `showcaseStylesheets`/`showcaseSetup`, the `viteConfig` option; all three root-only, D145, D148), imports its CSS as `?tss-css` (D144), calls `syncDocument` after applying `data-theme`, loads showcases through `loadShowcase(file, exportName)` and docs through `loadDoc(id)`, applies data and test updates under the one revision (D186).
- **Depends on:** C2 (data and live-channel shapes), C3, C5 (HTML shape), C8 (base and the module class), C15 (Vite helper).

### C11 — CLI `build` + `dev`

- **Goal:** orchestration: the pipeline's step 8 (D168). The CLI runs the check (C9), the tests and the routes, and calls the module's `build`/`dev` with lazy docs (D43, D187); it doesn't bundle anything itself.
- **Requirements:**
  - **The pipeline:** steps 1–7 from C9 → tests run through the runners (C7), results intersected with the in-scope tree, unlisted ones attached weakly, out-of-scope ones counted (D172, D186) → routes per node via `resolveRoute` → `module.build(ctx)` / `module.dev(ctx)` with the `Representation` and `DocsAccess` (C5). In `build`, test runs complete before the module's `build`; runners never overlap each other or other process-state work.
  - Theme resolution (several themes, D7); scoping all themes with C3's transform into a shell and a fonts stylesheet, written atomically to the CLI cache (`node_modules/.cache/tsshowcase/`), passed as `ctx.themeStylesheets`, regenerated in place on theme file changes in `dev` (D82).
  - **`build`:**
    - output to `build --out <dir>` (relative to the cwd), else config `outDir` (relative to the config file), default `./showcase` (D167);
    - **output guard:** refuse to write into a non-empty directory that tsshowcase didn't create (e.g. stdblox already has a `showcase/` source dir);
    - always checks; `--strict` parts (`check`, `tests`, `unavailable`, `unhandled`, `warnings`): `build` writes its output, then exits non-zero (D164); `--no-tests` (D197); logs the VCS ref its links use (D205); warns once when docs link to repo files while no VCS info is known (`unlinkedVcsLinks`, D110). The chunk plan decides whether test results are cached.
  - **Writing the output (D64):** the module returns the site as files; the CLI validates the paths, checks the checkable inclusion rules, and writes the output directory with the output guard, an atomic swap and a marker file (details: C2 plan, "Recorded for later chunks").
  - **Doc assets (D120, D187):** writes the assets of rendered docs only (`renderedAssets()`, once the module's `build` settles) next to the module's files (rejecting module files under `tss-assets/`; the no-absolute-paths check skips them); in `dev`, keeps the doc-assets directory in the CLI cache with every `DocRef`'s assets, rebuilt before each data update, passed as `DevContext.docAssetsDir`.
  - **`dev`:** the CLI watches files; structure changes rescan and re-list tests, `AnalysisResult.inputs` changes re-analyse, tracked reads re-verify (D177), doc changes re-render; each new snapshot (`DataUpdate`) and every test update (`TestsUpdate`, from the runners' watch mode, D18) carry one shared revision (D186). A change to the configured themes or to the showcase Vite config (D138) needs a restart (R9); root or package config changes, or a changed set of existing packages, restart `dev`; directory-config changes reload without a restart; `checkDocs` re-runs per snapshot; the runner set and run plan are frozen per session (C6 §11, C7 §5.1). `dev` never fails on issues.
  - Test error paths relativised (L2).
- **Depends on:** C3, C5, C6, C7, C9, C10.

### C15 — Vite pipeline helper (`-base-vite`)

- **Goal:** the reusable Vite build/dev pipeline that modules call (D43). Numbered C15 to avoid renumbering existing chunks.
- **Requirements** (detailed in the C15 plan, R9 and its redesign amendment):
  - API: `buildSite(ctx, app)` → `BuildResult` (`build.write: false`, D64) and `startDevServer(ctx, app)` → `DevSession`, where `app` (`ViteApp`) carries the module's entry, framework plugins and config, the user's showcase Vite config (D138), preludes (showcase stylesheets and setup), `selectDocs` (D210) and `prepareData(Representation)` keeping `revision` and `tests` (L20).
  - Vite runs with `root` at the repo; a set of **protected config keys** (`configFile: false`, `base: './'`, `appType: 'custom'`, `publicDir: false`, explicit `mode`/`define`, `build.write: false`, the virtual `index.html` input, no debug comments, …) is merged last and asserted on the resolved config, not re-applied (D146); `.env` files off by default (D139); `resolve.tsconfigPaths` on by default (D142); the user's copy of a duplicated plugin wins (D147).
  - Virtual modules: the data, the themes, the showcase loaders per file with showcasing components (D195), the rendered docs (one lazy chunk each, rendered through `ctx.docs` before the Vite build, D210) and the entry; a generated `index.html` with the colour-scheme meta and the inline selection script.
  - **Own CSS pipeline (D144):** theme stylesheets and module CSS (`?tss-css`) bypass Vite's CSS pipeline and the repo's PostCSS; `url()`s become asset imports; no `fs.allow` additions needed.
  - Dev server: HMR, a snapshot of the representation, its `DocsAccess` and the selected docs that survives reloads (test updates merged server-side), data and test updates pushed to the browser under one revision (D186), selected docs rendered on request from the current snapshot (D210), theme stylesheet updates as HMR of the themes module, `tss-assets/` served from `DevContext.docAssetsDir`; never sets `process.env` (D143).
  - **Runtime API (D89, H28):** framework-agnostic helpers that C10 (and other frameworks' modules) call: data and live updates (`getData`, `onData`, `onTestUpdate`, `applyTestUpdate`), `loadDoc(id)`, `loadShowcase(file, exportName)`, adopting the theme shell stylesheet plus module sheets into a given shadow root in order (with a `<style>` fallback, hot-updated in `dev`), the fonts stylesheet, theme state (`getInitialTheme()`, `setTheme()`, `subscribeTheme()`), and `syncDocument` (document-level `color-scheme`/canvas background, called by the module after it applies `data-theme`).
  - Output checks: single ES output, `index.html`, no `tss-assets/`, no absolute or machine-relative paths (candidates only from the build's own inputs, excused by rendered docs' raw sources, D146, `15d` QB).
  - Builds set `NODE_ENV=production` around the build (D94).
  - Framework-agnostic: no React knowledge.
- **Depends on:** C2, C3. (Rendered docs reach it only through C2's `DocsAccess`, so C5 is no edge.)

### C12 — End-to-end acceptance

- **Goal:** proof that v1 works as a whole.
- **Requirements:**
  - E2E matrix over the fixtures (D11): check / init / build / dev smoke, for single-package and workspace repos.
  - Built output opened in headless Chrome: navigation, showcase rendering, test results, theme switching. Local runs use Chromium only (no `install-deps`, no sudo); the three-engine matrix runs in CI.
  - **D89 acceptance:** a fixture whose showcase imports a global reset (incl. `html`/`body` rules) and a portal component: the shell is unaffected, the showcase is styled by the user's CSS, the portal is not clipped and paints above the shell; per-showcase scheme switching (`colorScheme` prop, `data-scheme` backdrop); doc anchor links scroll to the element; theme fonts load; a dark theme gives dark scrollbars and canvas.
  - **Contrast smoke test (D85):** computed WCAG AA contrast for the default themes' semantic pairs in the built output, incl. the severity colours and `--tss-link-broken`.
  - **Doc content (R7):** a doc image in `build` and in `dev` (updated when the image changes), footnote, cross-doc and author-anchor scrolling, themed alerts, broken and unlinked link states.
  - **`-base-vite` in a real browser (C15):** constructable-sheet adoption, theme asset URLs from a sub-path, theme HMR, the live channel and the hello, module CSS bypassing PostCSS, `hot.accept` from the published `dist/runtime` inside `node_modules`, doc chunks from a sub-path, docs re-loaded after a `dev` snapshot.
  - **The representation (D188–D191, D164):** directory configs, collections (globs), `paths`, `slug`, `order`, exemptions, `--strict` parts.
- **Depends on:** C11.

### C13 — Documentation

- **Goal:** user-facing and author-facing docs, consolidated and complete.
- **Requirements:**
  - Root README (what tsshowcase is, quick start, CLI reference).
  - Per-package READMEs completed.
  - The representation: its principles (D160) and conventions (name tests and showcases after the file they cover; `scope: ['src']`), the vouching effect of listings (D174), directory configs (forms, escaping, they execute code like configs), collections, exemptions, `vcs.ref` (D205), case rules (D201), migrating from an existing `showcase/` app (`--out`).
  - Focus-management libraries and Shadow DOM (the Fluent/Tabster worked example, D95); stacking of slotted showcase content under the shell's z-indexed elements.
  - Module authoring guide (incl. the isolation model, D89, `kinds`, `testRunners`, reserved rule namespaces, `FileInput` and concurrent `verify`), theme authoring guide (expanding "Building a theme that works with any module", D86), test-runner authoring guide (incl. `listTestFiles`); each points to the matching `@ofzza/tsshowcase-lint` preset (D88).
  - Interface changelog reviewed for the first `0.1.0` publish (D80).
- **Depends on:** C11 (can run in parallel with C12).

### C14 — Release

- **Goal:** v1 publishable (D31–D33).
- **Requirements:**
  - **Versioning (D80):** the first publish is `0.1.0`; the PR changeset check is switched on after it; PUBLISH.md documents the `0.x` major-changeset gotcha; decide whether `0.x` releases use a `next` dist-tag.
  - **CI extended:** Node 26 added to the `[22, 24]` matrix C1 already runs (D54); E2E (C12) in CI.
  - **Pack + install smoke test:** pack each package, install the tarballs into a scratch project, and run `check` and `build` there; the CLI's three schema copies (`config`, `package-config`, `directory-config`) at the package root and in its `exports`.
  - **Manual publish script** (no automated publish):
    - applies changesets versions;
    - runs the full CI suite and the smoke test;
    - publishes changed packages in dependency order (`-api` first, `-lint` after it), with scoped public access, skipping the private `-test`;
    - tags the release.
    - Supports a dry-run mode.
  - **Dogfood docs site (D41):** a GitHub Pages workflow that runs `tsshowcase build` on this repo. The structure already exists from C1 onwards.
- **Depends on:** C12, C13.

## C2 CLI duties → chunk (K10)

The C2 contract implies CLI-side behaviour beyond scanning and loading. Owners (R10: rewritten for the representation pipeline, D168; "shared" = a helper C9 implements and C11 reuses):

| Duty | Chunk |
| ---- | ----- |
| The scan: per-package in-scope trees, homes with roles, directory-config files found, `scan/*` issues (D169, D185, D175) | C4 |
| Read tracking for `RepoFiles`, per `verify` call (D177) | C4 |
| The `.git` reader behind `vcs.ref: CURRENT` and its errors (D205) | C4 |
| Resolving `vcs.ref` lazily on first use, cached per scan, thrown only when a VCS URL is produced (D205, `05b` Q1) | C4 |
| Producing the first VCS URLs, so an unresolvable ref fails early: `checkDocs` at the first static VCS link (D205) | C5 |
| Leaving `ref` and `fileUrlTemplate` out of the data's `VcsInfo` when the ref can't be resolved (D205) | C9 (shared) |
| Loading and validating directory configs after the scan, `config/several-configs` | C6 |
| Merging directory-config sources per directory (D188) and the other `config/*` issues | C9 (shared) |
| Loading test runners: candidates, pinning, detection, ids; `tests/runner-failed` for unpinned candidates that fail to load or whose `detect` throws, a thrown `PluginError` for a pinned runner that fails to load (D172, D209, C6 CC2) | C6 |
| Listing test files through the runners, intersected with the in-scope tree; one runner at a time, never alongside `analyze` or directory-config loading; `tests/no-runner`, listing failures (D172) | C9 (shared) |
| Validating `AnalysisResult` (out-of-scope keys dropped, nonexistent or duplicate keys thrown, undeclared kinds → `module/error`, D170) | C9 (shared) |
| Kind overrides (D189), collections (D190), exemptions (D191) | C9 (shared) |
| Associations (D173) | C9 (shared) |
| Building the representation: nodes, ordering, homes, doc ids, `PathNodeIndex` (D178, D179, D184, D185) | C9 (shared) |
| The check rule and `represent/*` issues (D174) | C9 (shared) |
| Synthetic `module/error` issues for undeclared kinds (D170), a throwing `verify` (C2 §5.5) or a module issue in a CLI-reserved namespace (C2 §9); invalid analysis keys are a thrown error instead | C9 (shared) |
| Issue placement, incl. `scan/*`, `docs/*`, `config/*`, deriving `Issue.package` (D180) | C9 (shared) |
| Parsing `--rule` and validating `rules` keys against the rule-id table (D175, D206) | C6 |
| Applying severity overrides with patterns, per-package `rules` and `--rule` (D176, D206) | C9 (shared) |
| The argv normaliser for `--strict`, `--no-strict` and `--rule`, in command-line order (D207) | C6 |
| The `--strict` parts `check` and `warnings` (D164) | C9 (shared) |
| The `--strict` parts `tests`, `unavailable` and `unhandled` (D164) | C11 |
| Reserving the docs step in `check` (D123) | C9 |
| Wiring the docs step to `checkDocs` over every `DocRef` (D183) | C5 |
| Running tests in `build` and `dev`, intersection, weak attachment, `outOfScope`, `Representation.tests`, `TestsUpdate` streaming (D172, D186) | C11 |
| Relativising `TestError` paths (L2) | C11 |
| Routes per node via `resolveRoute` (D183) | C11 |
| `createDocs`: lazy rendering, `assetsOf`, `renderedAssets()` (D187) | C5 |
| A `DocsAccess` per snapshot; writing the assets of rendered docs only; the `dev` doc-assets directory with every `DocRef`'s assets (D187) | C11 |
| Writing `ScaffoldResult` files (`ifExists`, never overwriting silently, `notes` printed; C2 §5.10, D64, D196) | C9 |
| Writing `BuildResult.files` (D64): path validation, inclusion checks, output guard, atomic swap, marker file, module files under `tss-assets/` rejected | C11 |
| `SOURCE_DATE_EPOCH` for `generatedAt` | C11 |
| One shared `revision` for snapshots and test updates (D186) | C11 |
| Shutdown order in `build` and `dev` (abort signals → `DevSession.close()` → `dispose()`) | C11 |
| `dispose()` after `check` and `init` | C9 |
| `dev` invalidation: rescans and re-listing on structure changes, re-analysis on `AnalysisResult.inputs` changes, re-verify on tracked reads, directory-config reloads (D177) | C11 |
| Theme selection, duplicate names, scoping, CLI cache and atomic writes (D82) | C11 |

## Dependency edges

| Chunk | Depends on                                           |
| ----- | ---------------------------------------------------- |
| C1    | –                                                    |
| C2    | C1 (placement only)                                  |
| C3    | C2 (theme manifest; fills the `-api` catalogue, D28) |
| C4    | C2                                                   |
| C5    | C2, C3, C4 (implemented after C9: the representation, `PathNodeIndex`, D194) |
| C6    | C2, C4 (workspace detection, scanner inputs; K9)     |
| C7    | C2, C4, C6                                           |
| C8    | C2, C4 (fixtures)                                    |
| C9    | C4, C5 (planning only: the docs step's API; C5 is implemented after C9 and wires it), C6, C7, C8 |
| C10   | C2, C3, C5, C8, C15                                  |
| C11   | C3, C5, C6, C7, C9, C10                              |
| C12   | C11                                                  |
| C13   | C11                                                  |
| C14   | C12, C13                                             |
| C15   | C2, C3                                               |

## Planning order (stage 3)

Plans are approved one at a time. "Parallel" means a group's members only depend on chunks already planned, so they could be planned in any order or at the same time.

| Wave | Chunks                                          | Why                                                                                                                                           |
| ---- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| P1   | **C1**                                          | Small; fixes the layout everything else refers to.                                                                                            |
| P2   | **C2**, then **C3**                             | C2 is the critical, collaborative contract. C3 fills C2's catalogue, so the C2 plan stays open until C3 is approved.                          |
| P3   | **C4, C5, C6, C7, C15** (approved one at a time, in any order, except **C4 before C6**) | Each depended only on C2/C3 when planned (C6 also on C4's seam, K9); C5 → C4 (R7) and C7 → C4, C6 (R10) were added after their plans were approved. Planning C7 early also explores the Vitest Node API, the least-known piece; since C2 is approved, a finding there means backtracking to C2. |
| P4   | **C8**                                          | Needs C4's fixtures. Its round-2 planning led to the representation redesign (`08b`), applied to every approved plan first (plans 1–10); the C8 plan is then rewritten (plan 11). |
| P5   | **C9** and **C10** (parallel)                   | C9 needs C4, C5, C6, C7 and C8; C10 needs C3, C5, C8 and C15. The loadable module class lives in C8, so they don't depend on each other.          |
| P6   | **C11**                                         | Integrates C9 and C10.                                                                                                                        |
| P7   | **C12** and **C13** (parallel)                  | Both depend only on C11.                                                                                                                      |
| P8   | **C14**                                         | Final.                                                                                                                                        |

## Implementation order (stage 4)

`C1 → C2 → C4 → C6 → C7 → C8 → C9` gives a **usable `check` + `init` early** (D194: `check` lists test files, so C7 comes before C9). The point (D29) is to **lock down the inputs** (scanning, analysis, the representation) against real code before they are shaped into a themed showcase app. C9 reserves the docs step that C5's implementation wires to `checkDocs`.
Then `C3 → C5 → C15 → C10 → C11 → C12 → C13 → C14`. C3's implementation also extends `-api` with the catalogue constants.

If implementation reveals a contract gap, we backtrack to the C2 plan, edit it (in place before the first publish; with an `-api` changeset after it, D80), and re-approve.
