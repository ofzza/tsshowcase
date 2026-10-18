# Super-plan — tsshowcase v1

> Status: **APPROVED** (after a final consistency review, see `01a-super-plan-review.md`); revision R1 **re-approved** together with the C2 plan (2026-10-18); revision R2 **re-approved** together with the C3 plan (2026-10-18); revision R3 (checkpoint review and prototype spikes) **re-approved** (2026-10-18); revision R4 (review of all plans before wave P3) **re-approved** (2026-10-18); revision R5 (C4 planning) **re-approved** together with the C4 plan (2026-10-18). Decisions referenced as `D<n>` are in AGENTS.md §3.

## Revisions

- **R1 (during C2 planning, from the C2 review `02a-c2-plan-review.md`, F37–F40):** text brought in line with decisions made after the super-plan's approval, plus chunk duties nobody owned.
  - C1: Node floor `>=22.12` (D49), CI matrix `[22, 24]` from C1 (D54).
  - C2: `showcases[]` (D61); only `analyze` required (D63); context-object signatures; module options as JSON Schema (D58); repo-level verification is `analyze` returning issues; `build` returns files (D64); config moved out (D66).
  - C5: `Doc.headings`. C6: per-package config files (D59). C11: the D64 write duties. C14: CI adds Node 26 (D54).
  - Added after the C2 group B decisions: `resolveRoute(target, ctx, heading?)` and routes in the data (D67); `reportTestUpdate` (C10); `init <dir>` (C9); plain-text docs and link mapping in C5 (D71, D73); merged per-package options in C6 (D69).
  - After the remaining C2 decisions: data embedding is the module's business (D77, C2 bullet updated); `check` is a prerequisite of `build`/`dev` (D76, C11); `loadTestRunner` candidates (D74, C6); module-defined CLI flags (D79, C6/C9); group READMEs (D78, C11); versioning from `0.1.0` (D80: C1 amendment A2, C14).
- **R2 (during C3 planning, D81–D89, plus the C3 review `03a-c3-plan-review.md`):**
  - Packages: new published `@ofzza/tsshowcase-lint` (D88); the catalogue reference docs move from `-theme-default` to `-api` (D83); `-theme-default` gains `css/` and named file exports; the CLI gains the theme scoping transform.
  - **Isolation model (D89):** the shell lives in a shadow root (theme and module CSS adopted there), showcases are light-DOM children slotted into their stage, receive a colour scheme through a module-defined interface, and each gets a light/dark selector; regions, `@scope` and `tss-plain` are gone (D87 superseded).
  - C3: no `@layer` (D81); the CLI scopes theme CSS into a shell and a fonts stylesheet (D82, transform implemented in C3); themes may style anything inside the shell within three rules (D86); states via ARIA/data attributes (D84); contrast is a review guideline (D85); C3 adds the `-lint` package.
  - C5: catalogue classes and `tssData`. C10: the shadow-root shell, slotted showcases, scheme selectors, the shell opt-in for D41 showcases, `-lint` presets. C11: the CLI cache, atomic writes, calls the transform, passes `themeStylesheets` (new edge C11 → C3). C13: `-lint` and the theme recommendation. C14: publishes `-lint` after `-api`. C15: adopts the shell stylesheet into the shadow root, adds the fonts stylesheet, `fs.allow` for the cache and theme assets, hot-reloads adopted stylesheets.
  - Stale text fixed: C2 section (`themeStylesheets`, inclusion, changelog from `0.1.0`), C13 changelog, C1's package list.
  - After the second C3 review (`03b-c3-plan-review.md`): C15's runtime API and C10's isolation duties made explicit (H28); C12's D89 acceptance and contrast smoke test (H33, H31); the `-base-vite`/`-base-react` rows and C10's showcases (H30).
- **R3 (checkpoint review before wave P3, `01b-checkpoint-review.md`, D91–D93):**
  - New "C2 CLI duties → chunk" checklist (K10); edge C6 → C4 and the C4 scanner/workspace-detection seam, C4 planned before C6 (K9); theme resolution split between C6 and C11 (K14).
  - C10: React fallback for repos without React (D91). C11: `build [outDir]` (D93), `check` in the data assembly (K25). C5: group targets (D78).
  - Stale text: C2 bullet (`onThemeStylesheetsUpdate`, `dispose`), C15 `dev()` line, C11 "Depends on", the P3 rationale, the closing note on contract gaps (K12).
  - Open points for every remaining chunk now live in AGENTS.md ("Open points per chunk"). A "v1 cut line" review runs once all chunk plans are approved, before implementation (K21).
  - Prototype spikes instead of gates (K17/K20, `01c-spike-results.md`): the in-process Vitest runner is proven (C7, D94) and the D89 technique is confirmed in Chromium, Firefox and WebKit, with fixes (C10, C13, C15, D95).
- **R4 (review of all plans before wave P3, `01d-pre-p3-review.md`, D96–D102):**
  - C8: a preliminary React showcase file format (D96). C10: finalises it; theme-owned breakpoints (D97); React resolved per showcased package (refines D91).
  - L1–L3 recorded as known open issues (AGENTS §4); the rest as open points per chunk. The cut-line review stays after all plans (D98); C10 stays one chunk (D99); shell-mode showcases get no scheme selector (D100; C2 rule 7, C3 §1); `include` folders and the marker-file skip (D101; C2 §5.5, §10, C4, C6); `BuildContext.themes` carries only name, label and scheme (D102, C2 §5.8).

- **R5 (C4 planning, D103–D110, reviews `04a-c4-plan-review.md`):**
  - **What exists (D103, D109):** hard ignores, then config `ignore` (renamed from `exclude`), then `.gitignore` and the `.tsshowcaseignore` alias, at or below the root only; git's parent rule for every source; no `.git/info/exclude`; no git calls.
  - **Renames (D107):** `include` → `scope` (where to look, never un-ignores), `exclude` → `ignore`; scanner inputs `{ scope, ignore, skippedPackages, outDir }`.
  - **Access (D108):** `RepoFiles`/`DirectoryInput` treat ignored paths as nonexistent unless `{ ignored: true }`; `.git`/`node_modules` never reachable (C2 amendment B4).
  - **C8 → C9 (D106):** units (and `analyze` issues) outside `scope` are dropped by the CLI, not the module.
  - **VCS (D110):** from `package.json#repository` and config `vcs.*` only, links always use `HEAD`; no git (revises D24, D105).
  - **New private package `-test` (D104)** for test helpers (C1 amendment A3).

## Packages (D14, D15, D26, D27, D43)

| Package                                  | Role                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@ofzza/tsshowcase-api`                  | Contracts only: TS types and interfaces, `IMPLEMENTED_API_VERSION`, interface changelog, small runtime helpers (e.g. issue constructors). Has no heavy deps.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `@ofzza/tsshowcase`                      | The CLI (binary `tsshowcase`): repo scanning, markdown pipeline, plugin loading, config, data assembly, theme scoping (PostCSS, D82), `dev` file watching, and the `check` / `build` / `dev` / `init` actions. It doesn't bundle anything itself; modules build (D43). Depends on the built-in packages so they can be looked up by name (D9).                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `@ofzza/tsshowcase-theme-default`        | Default themes: built-in names `default-light` and `default-dark`; CSS in `css/` (`files: ["dist", "css"]`), named file exports for composing themes, user-facing theming docs. The catalogue reference docs live in `-api` (D83).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `@ofzza/tsshowcase-lint`                 | Published lint rules, plugins and presets for tsshowcase plugins (D88): a stylelint plugin and an ESLint plugin enforcing D46/D86, presets per plugin kind (theme CSS, module CSS, module TSX). Enforced on this repo's own theme and module packages; opt-in for third-party authors. Contains no code that runs linting. |
| `@ofzza/tsshowcase-testrunner-vitest`    | Prepackaged Vitest test runner (D3), with run and watch modes. Built-in name: `vitest`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `@ofzza/tsshowcase-base-vite`            | Reusable Vite pipeline helper (D43). Not a loadable module. Handles: data embedding as a virtual module, a framework-agnostic runtime API for theme stylesheets (adopted into a given shadow root, fonts stylesheet in the document, dev updates) and theme state (D89), `base: './'`, in-memory build output (returned as files for the CLI to write, D64), dev server, HMR and the live update channel. Modules call it with their framework plugins and app entry.                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `@ofzza/tsshowcase-base-react`           | Shared React base (D20, D26, D48). Layout-agnostic and **not itself a loadable module**. Two halves:<br>• **check/init (Node, no bundler):** static entry analysis, base rules and the `headless` preset, scaffold templates;<br>• **build/dev:** a **React wrapper around `-base-vite`** (plugin-react, React deduped from the showcased repo, tsconfig paths) plus the runtime building blocks (data and live-channel access, the shadow-root shell host, slotted showcases with per-showcase scheme selectors, README HTML renderer, test-results view, theme switcher, hash router with in-shadow heading scroll; D89).<br>Exposes a **base module class** with analyze / verify / scaffold / runTests / build / dev already implemented. Layout modules extend it and supply only their app entry and layout. Depends on `-base-vite`. |
| `@ofzza/tsshowcase-test`                 | **Private, never published** (D104): test helpers for this repo's own tests (`copyFixture()`, `fixturePath()`); a devDependency only; skips itself from the dogfood check (`skip: true`). Added by C4 (C1 amendment A3). |
| `@ofzza/tsshowcase-module-react-default` | The v1 rendering module (D22). Built-in name: `react-default`. Contains **layout only**: app shell, navigation tree (repo → packages → units), route scheme, and the home/package/unit pages composed from the base building blocks.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

## Chunks

Each chunk lists **Goal**, **Requirements** and **Depends on**. Each chunk delivers its own docs, its tests and the fixtures it needs (D10 conventions; docs ship with code).

### C1 — Monorepo foundation

- **Goal:** an empty but fully wired workspace.
- **Requirements:**
  - npm workspaces with the package skeletons listed above (except `-lint`, added by C3, D88, and the private `-test`, added by C4, D104).
  - TS (ESM, Node `>=22.12`, D17/D49), build via `tsc` project references.
  - Prettier (160), ESLint, Vitest workspace config, `npm run ci` aggregate script.
  - A `fixtures/` directory convention. Fixtures are **isolated from the workspace**: excluded from `workspaces`, from root lint/test/type-check, and from dependency hoisting (each fixture has its own install where it needs `react`/`vitest`). Otherwise "resolve React from the showcased repo" can't be tested.
  - **Basic CI (D32):** a GitHub Actions workflow that runs lint, type-check, test and build on push/PR on a Node `[22, 24]` matrix (D54), so every chunk is checked.
  - changesets set up for independent versioning (D31).
  - The D10 conventions documented in AGENTS.md.
  - **Dogfooding (D41):** the package skeletons follow the unit convention (a dir per exported unit, README, tests, showcases per D37).
- **Depends on:** nothing.

### C2 — Plugin contracts (`-api`) ★ critical path, designed collaboratively in detail before any code (assignment requirement)

- **Goal:** the complete, documented v1 contracts.
- **Requirements:**
  - **Input data model:**
    - repo info (root, VCS host; links use `HEAD`, D110);
    - packages[] (D16, D19), each with: package info (sourced from package.json in v1, but source-agnostic so other repo types can come later), licence (D12), README raw and HTML (D23), unit hierarchy;
    - how the repo root README/licence relate to package READMEs/licences, including the single-package collapse (no duplicated README on home and package page);
    - per unit (a directory exporting 1+ symbols; nesting allowed, D36, so a unit can be both a page and a parent of sub-units): kind, exported symbols, README raw/HTML, tests (hierarchy plus outcomes), a possibly empty list of showcase entries (D37, D61);
    - extensible: additive only (D13).
  - **Rendering module interface** (default-exported class, D9 loading). Only `analyze` is required; every other method is optional and feature-detected per action (D63). Methods take context objects:
    - `apiVersion` (semver range, D13);
    - `analyze(ctx)` → exported units, plus repo-level issues (D5);
    - `verify(dir, ctx)` → `{ ignored } | { issues[] }` (D2), receiving the file list plus lazy, cached accessors (D40);
    - `runTests(ctx)` (D3, D34, D42): the module calls `ctx.loadTestRunner(ref?)` (an explicit ref from config or the module default, otherwise auto-detection), runs or watches, maps the raw results to units, and returns them. In dev, it keeps pushing test updates to the CLI through `ctx` (D18);
    - `resolveRoute(target, ctx, heading?)` (D35, D67): the in-app URL for home, a package or a unit; the CLI stores routes in the data;
    - `build(ctx)` and `dev(ctx)` (D43): the module builds the static app and returns it as files for the CLI to write (D64), or runs a dev server. `ctx` carries the assembled data, the themes (names, labels, schemes), the CLI-scoped `themeStylesheets` (D82) and (in dev) `onDataUpdate` / `onTestUpdate` / `onThemeStylesheetsUpdate` subscriptions;
    - `scaffold(ctx)` for `init` (D16): returns files for the CLI to write (D64);
    - `dispose()` at the end of every action (D63);
    - module-specific options schema, as JSON Schema (D6, D58);
  - **Inclusion rules for any module's output** (D43): how the CLI-scoped theme stylesheets are included (D82), how the switcher attribute works, showcase isolation (D89), static/relocatable output with no leaks. Mandatory even for modules that don't use `-base-vite`. How the data is embedded is the module's business (D77).
  - **Issue model:** severity, message, file, rule id.
  - **Test-runner interface:** `apiVersion`, `detect(repo)`, `run(options)`, `watch(options, onUpdate)`; a raw result hierarchy (file → suite → test, with status, duration and error); never throws on failing tests (D8, D42).
  - **Theme contract:** a theme manifest (`apiVersion`, name(s), CSS entry files, light/dark flags, D7).
  - **Theme catalogue constants (D28):** every documented CSS class name and CSS variable name, exported as typed constants that modules import. The _values_ and the _design_ of the hierarchy come from C3; C2 provides the slot and the export mechanism, and C3's plan fills it in (it extends `-api`).
  - **Data handed to `build`/`dev`:** its shape, plus the dev update protocol (data changes and test updates, D18).
  - **Repo-level verification:** issues not tied to one directory (e.g. an unresolvable entry, an export whose source is shared with other exports), returned by `analyze`.
  - **Output tolerance:** the CLI tolerates unknown fields in module outputs (analysis results, issues, test results) as well as in inputs (D13).
  - Interface changelog: `-api`'s changesets `CHANGELOG.md`, starting at `0.1.0` (D80).
- **Depends on:** C1 (only for where the code lives; planning is independent).
- **Note:** the C2 plan stays _open_ until the C3 plan is approved, because C3 fills in the catalogue constants in `-api` (D28). Implementing C3 modifies the `-api` package.

### C3 — Theme system + default themes

- **Goal:** design the CSS variable hierarchy and class catalogue (the theme ⇄ module contract; published as constants in `-api`, D28), plus the light and dark themes.
- **Requirements:**
  - **Layered variables:**
    - base tokens (primary/secondary colours, base spacing, base font size, radius, …);
    - → derived scales (palette ramps, spacing scale, type scale) computed from the base tokens (`calc()`, `color-mix()`);
    - → semantic tokens (surface, text, link, border, code background, success/failure, …);
    - → component tokens.
  - Overriding any layer cascades to the layers below it.
  - Documented, `tss-`-prefixed class catalogue (states as ARIA/data attributes, D84): layout shell, nav, headings, paragraphs, links, lists, tables, code (inline and block), blockquote, showcase frame and error, test-result list and status, theme switcher.
  - Covers the markdown HTML output (C5).
  - Runtime theme switching with a `prefers-color-scheme` default (D7). Persists the user's choice and avoids a flash of the wrong theme.
  - **Isolation model (D89):** specified here (shadow-root shell, slotted light-DOM showcases, inheritance resets, colour scheme for showcases); implemented in C10/C15.
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

- **Goal:** turn a repo on disk into the repo / package / directory layer of the C2 input model. Units come from the module's `analyze` (C8), tests from the runner (C7), and HTML from C5.
- **Requirements** (details: the approved C4 plan):
  - **What exists (D103, D109):** hard ignores (`.git`, `node_modules`, an explicitly configured output directory, any directory holding tsshowcase's output marker file, skipped packages); then config `ignore` patterns (gitignore syntax incl. `!`, overriding ignore files on the same path); then `.gitignore` files and their `.tsshowcaseignore` alias at or below the root only, with git's semantics (no `.git/info/exclude`, no global excludes, tracking state irrelevant, no git calls). Git's parent rule holds for every source. Behaviour is the same for non-git directories.
  - **`scope` (D101, D107):** paths and glob patterns (default the whole repo) saying where to look: the directories checked and showcased. It never un-ignores. Repo metadata is read wherever it lives. Units and `analyze` issues outside it are dropped by the CLI (D106, C9).
  - Lazy file content accessors; `read`/`exists`/`list` treat ignored paths as nonexistent unless `{ ignored: true }` (D108).
  - Workspace detection (npm, yarn and pnpm workspaces); single package = one package (D16). **A separate function** that C6 calls first, to find per-package config files (K9).
  - **The scanner takes its exclusions as inputs:** `{ scope, ignore, skippedPackages, outDir }`, resolved by C6 from the config (K9, D107).
  - Read tracking for `RepoFiles` / `DirectoryInput` (D40), so `dev` can invalidate (K10).
  - package.json info, licence detection (D12), root and package READMEs.
  - **VCS info (D24, D110):** from the root `package.json#repository`, overridden by config `vcs.*` and flags; links always use `HEAD`; no git.
  - Fixtures: a single-package repo, workspace repos, and broken variants; the `copyFixture()` helper in the private `-test` package (D104).
- **Depends on:** C2.

### C5 — Markdown pipeline (CLI core)

- **Goal:** markdown → themed HTML (D23).
- **Requirements:**
  - unified/remark/rehype with GFM and heading slugs.
  - Emits only C3 catalogue classes (and `tssData` attributes where needed).
  - **Link rewriting:**
    - unit, package and group READMEs (and their directories) → in-app URLs via the module's `resolveRoute` (D35, D73, D78); a group without a route → the VCS host;
    - repo file → host URL template (GitHub/GitLab/Bitbucket, overridable, D24);
    - external links left untouched;
    - heading anchors (`#slug`, in-page or cross-README) rewritten via `resolveRoute(target, ctx, heading)` (D35, D67), since bare `#slug` collides with hash routing.
  - Collects each document's headings (depth, text, id) for `Doc.headings` (C2).
  - Plain-text docs (`LICENSE`, `.txt`; `Doc.format: 'text'`, D71) render as escaped `<pre>` with a catalogue class.
  - Link-to-target mapping per D73 (a unit's README or directory → the unit, …, anything else → the VCS host).
  - Syntax highlighting for code blocks (D39), emitting only themable token classes, with no inline colours.
- **Depends on:** C2, C3 (class names).

### C6 — Plugin loading + configuration (CLI core)

- **Goal:** resolve and validate modules, themes and runners (runners on the module's behalf, D34), and the effective config. For themes, C6 resolves references (D9, `#name`), loads and version-checks theme sets; C11 selects the configured themes, checks duplicate names and `defaultTheme`, and scopes (K14).
- **Requirements:**
  - Reference resolution order: built-in name → npm package → path (D9).
  - TS loading via jiti.
  - Default-export class instantiation.
  - `apiVersion` semver check with clear errors (D13).
  - `ctx.loadTestRunner(ref?, { candidates? })` helper: resolves an explicit ref, or auto-detects among candidate runners; throws on ambiguity, `undefined` when none (D42, D74).
  - Module-defined CLI flags mapped onto option paths (`optionFlags`, D79): parsing from the schema, precedence over config, collisions, `--help`.
  - The config schema (D6, D58, D59, D62), owned by the CLI package, not `-api` (D66): `config.schema.json` / `package-config.schema.json`, inferred types, `defineConfig`, documented in the CLI README. Design starting point: C2 plan §10.
  - Config file discovery (`tsshowcase.config.ts|js|json`), validated against that schema, merged with CLI flags (D6).
  - Per-package config files in workspace packages (D59): discovery (using C4's workspace detection), the restricted package schema, and merging over the root config; each package's merged `moduleOptions` validated against the module's `optionsSchema` and handed over as `PackageInput.options` (D69).
  - Validation of module options.
  - **Scan and VCS keys:** `scope` and `ignore` (D107) with repeatable `--scope` / `--ignore` flags, `vcs.webUrl` / `vcs.directory` / `vcs.host` / `vcs.fileUrlTemplate` with `--vcs-*` flags (D110); resolved into C4's scanner inputs.
- **Depends on:** C2, C4 (workspace detection and scanner inputs, K9).

### C7 — Vitest test runner (`-testrunner-vitest`)

- **Goal:** the D3 prepackaged runner.
- **Requirements:**
  - Runs the showcased repo's Vitest via the Vitest Node API, using the repo's own vitest install and config.
  - Returns the **raw** C2 test hierarchy (file → describe → test, with status, duration and error message). Mapping results to units is the module's job (D42), not the runner's.
  - Watch mode that streams updates (D18).
  - `detect(repo)`: vitest in deps, `vitest.config.*`, a `test` block in `vite.config.*`, workspace/projects config (D42).
  - A failing test never throws (D8).
  - **Proven by the spike (`01c-spike-results.md`, D94):** `createVitest` + the reporter API (`onTestRunStart`, `onTestModuleEnd`, `onTestRunEnd`), `vitest/node` resolved from the repo root via `createRequire`; supported range `^3.2 || ^4 || ^5`; restore `process.env` and `process.exitCode` after each run; "no tests" = completed; results carry `project` and `unhandledErrors`; no port/HMR isolation needed next to our Vite.
- **Depends on:** C2.

### C8 — React base: Node side (`-base-react`)

- **Goal:** everything `check` and `init` need for React/TS repos. Also the first version of the React **base module class** (Node-side methods: analyze / verify / scaffold), **plus the loadable `-module-react-default` class** extending it, so that C9 can run `--module react-default` before C10 exists. No bundler involvement (D48).
- **Requirements:**
  - **Static entry analysis** with the TS compiler API (D5):
    - package.json `exports`/`main`/`types` → entry → follows `export *` and named re-exports → source directories;
    - maps built entries (`dist/…`) back to source (e.g. the `source` export condition, tsconfig `rootDir`/`outDir`, or a config override), since `dist` may not exist;
    - uses the showcased repo's own TypeScript if it has one;
    - exports whose source lies outside their own package (npm dependencies, sibling workspace packages) are external re-exports, never units (units outside `scope` are dropped by the CLI, not the module: D106);
    - classifies exports by kind (component / hook / function / class / constant / type), as needed by the D37 showcase rules.
  - **Base rules (D21):**
    - every exported unit has its own directory;
    - every unit has a README; every non-type-only unit has a test file (D37, D38);
    - a showcase is required per kind: components and hooks yes, functions/classes/constants optional, type-only not applicable (D37);
    - a showcase, when present, default-exports a component;
    - non-unit directories are ignored.
  - Opt-in `headless` preset (D47): `useX` holds the logic and prop getters, `renderX` is a pure presentation function, and `X` only wires the two.
  - Rule ids and configurable severities.
  - Scaffold templates (README, test, showcase stubs). The templates also keep showcase files out of the user's library build and published package.
  - **A preliminary showcase file format (D96):** naming, the default-export signature (`{ colorScheme }`), the `shell` opt-in, several showcases per unit; C10 finalises it.
- **Depends on:** C2, C4 (fixtures).

### C9 — CLI shell + `check` + `init`

- **Goal:** the first usable vertical slice.
- **Requirements:**
  - CLI framework, global flags (`--module`, `--theme`, `--config`, `--cwd`).
  - **`check`:**
    - tree report of every scanned directory (in `scope`, not ignored), with per-directory issues (human-readable, coloured);
    - `--format json`;
    - non-zero exit code on errors (D16).
  - **`init <dir>`** calls the module's `scaffold` (and reports clearly when a module lacks it, per D13 feature detection).
  - **Self-check (D41):** `tsshowcase check` runs on this repo in CI.
- **Depends on:** C4, C6, C8.

### C10 — React runtime building blocks + default module (`-base-react` runtime, `-module-react-default`)

- **Goal:** the generated SPA. The base module class gains `build`/`dev`, implemented as a React wrapper around `-base-vite` (C15), plus the runtime building blocks. `-module-react-default` adds only its app entry and layout (D48).
- **Requirements:**
  - **React input to `-base-vite` (C15)** and `build`/`dev` implementations that call it:
    - `@vitejs/plugin-react`;
    - react/react-dom **resolved and deduped from the showcased repo**; a repo without React gets a React bundled with `-base-react` (D91) (pitfall: duplicate React). The dedupe also covers module runtime files living in the CLI's `node_modules`, and the module's packages must not pull in their own React copy through peer auto-install;
    - respects the repo's tsconfig paths;
  - **Base class `runTests`** (D42): `ctx.loadTestRunner` → run / watch → map raw results to units (by unit directory, incl. nested units, D36) → return / push via `ctx.reportTestUpdate`.
  - **App:**
    - hash router;
    - nav tree (repo → packages → units, D19; nested units are both pages and parents, D36);
    - home page = repo README; package pages with README and licence; unit pages with README HTML, the unit's showcases (zero or more, D37/D61) each in Suspense and an error boundary, and test results;
    - `resolveRoute` implementation (D35);
    - theme switcher UI component (persistence and selection live in C15).
    - showcases of the `-base-react` components themselves (dogfooding, D41), using mock data.
  - **Isolation (D89, C3 §1):** the `<tsshowcase-app>` host and its shadow root (theme shell stylesheet adopted first, then module CSS, via C15's runtime API; `:host { all: initial !important }`; host as a stacking context; document scrolling, no clipping on stage ancestors); each showcase portalled into a light-DOM `[data-tss-showcase]` wrapper assigned to a named slot in its stage (`slot { all: initial !important; display: contents !important; color-scheme }`); the `showcaseStylesheets` and `showcaseSetup` module options (document-level CSS for app typography, H1; a setup module run before showcases mount, e.g. Fluent's Tabster Shadow-DOM API, D95); `isolation: isolate !important` on the host and `!important` on every declaration after `all: initial !important`; no shell handlers reacting to synthetic events bubbled from showcases; every showcase gets a `colorScheme` prop (the React showcase interface) and its own light/dark selector (follows the theme until changed, reset on theme switch, not persisted) with `data-scheme` on the stage; a showcase can opt into rendering inside the shell (D41 showcases); shell overlays inside `.tss-shell`; `data-theme` rendered from C15's theme state; the router scrolls to and focuses route-targeted headings inside the shadow root (`aria-current="location"`); key events from shell inputs stopped at the host, the stage labelled from inside the shadow tree; the smoke render passes a scheme and feature-detects constructable stylesheets (`<style>` fallback). Before locking the technique, a throwaway browser spike is recommended while planning C10/C15.
  - **Responsive layout (D97):** breakpoints live in theme CSS, which overrides layout tokens in its own media queries; module CSS has no length-based media queries. The needed tokens and missing catalogue classes come as a C3 amendment.
  - **Only C3 classes and variables are used** (plus own-prefixed arrangement classes, D86); passes the `-lint` presets (`stylelintConfigs.module`, `eslintConfigs.module`); `tssData` for states.
  - Consumes the embedded data and the dev live channel provided by `-base-vite` (C15).
- **Depends on:** C2 (data, test-result and live-channel shapes), C3, C5 (HTML shape), C7 (runner, for `runTests`), C8 (base and the module class), C15 (Vite helper).

### C11 — CLI `build` + `dev`

- **Goal:** orchestration. The CLI assembles data and calls the module's `build`/`dev` (D43); it doesn't bundle anything itself.
- **Requirements:**
  - **Data assembly:** C4 scan → module `analyze` → `verify` × every directory (the `check` prerequisite, D76) → module `runTests` (which uses C7) → C5 HTML (links via `resolveRoute`) → `module.build(ctx)` / `module.dev(ctx)`.
  - Theme resolution (several themes, D7); scoping all themes with C3's transform into a shell and a fonts stylesheet, written atomically to the CLI cache (`node_modules/.cache/tsshowcase/`), passed as `ctx.themeStylesheets`, regenerated in place on theme file changes in `dev` (D82).
  - **`build`:**
    - output to the positional `build [outDir]` (relative to the cwd), else config `outDir` (relative to the config file), default `./showcase` (D93);
    - **output guard:** refuse to write into a non-empty directory that tsshowcase didn't create (e.g. stdblox already has a `showcase/` source dir);
    - `--fail-on-test-failure` (D8); `--no-tests` (optional). The chunk plan decides whether "tests unavailable" (D42) also fails in strict mode, and whether test results are cached.
  - **Writing the output (D64):** the module returns the site as files; the CLI validates the paths, checks the checkable inclusion rules, and writes `--out` with the output guard, an atomic swap and a marker file (details: C2 plan, "Recorded for later chunks").
  - **`check` first (D76):** `build` and `dev` run the full `check` and print the report; errors fail `build` unless `--no-check` (config `check: false`); `dev` never fails on them.
  - **Group READMEs (D78):** a `README.md` in a group directory becomes `GroupNode.readme`.
  - **`dev`:** the CLI watches files. README changes are re-rendered, and adding or removing a unit or file triggers re-analysis. The new data is pushed to the module's running dev server (`ctx` update subscription). Test updates come from the module's watch mode (D18, D42).
- **Depends on:** C3, C5, C6, C7, C9, C10.

### C15 — Vite pipeline helper (`-base-vite`)

- **Goal:** the reusable Vite build/dev pipeline that modules call (D43). Numbered C15 to avoid renumbering existing chunks.
- **Requirements:**
  - API: `build({ data, themes, plugins, entry, resolve… })` → files (`build.write: false`, D64) and `dev({ …, onDataUpdate, onTestUpdate, onThemeStylesheetsUpdate })`.
  - Data embedding as a virtual module; adopting the CLI's shell stylesheet into the shell's shadow root (constructable stylesheets, with hot reloading of adopted sheets in `dev`) and adding the fonts stylesheet to the document (D82, D89; no scoping here); the theme selection setup (`data-theme` on `.tss-shell`, `prefers-color-scheme` default, persisted choice), per the C2 inclusion rules.
  - `base: './'`; `server.fs.allow` for module runtime files, the CLI cache directory (theme stylesheets) and theme asset directories outside the repo root.
  - Dev server, HMR, and pushing data and test updates to the browser (D18).
  - **Runtime API (D89, H28):** framework-agnostic helpers that C10 (and other frameworks' modules) call: adopt the theme shell stylesheet plus module sheets into a given shadow root in order (with a `<style>` fallback), add the fonts stylesheet to the document, apply dev stylesheet updates (`replaceSync()` from `DevContext.onThemeStylesheetsUpdate`, forwarded over the live channel), theme state (`getInitialTheme()`, `setTheme()`, `subscribe()`), and the document-level `color-scheme`/canvas background plus the `index.html` meta and inline selection script.
  - Theme stylesheets are processed as toolchain inputs (`?inline`-style, emitting their assets; never `?raw`).
  - Builds set `mode`/`NODE_ENV=production` explicitly (a test run may leave `NODE_ENV=development` behind; D94).
  - Framework-agnostic: no React knowledge.
- **Depends on:** C2, C3.

### C12 — End-to-end acceptance

- **Goal:** proof that v1 works as a whole.
- **Requirements:**
  - E2E matrix over the fixtures (D11): check / init / build / dev smoke, for single-package and workspace repos.
  - Built output opened in headless Chrome: navigation, showcase rendering, test results, theme switching.
  - **D89 acceptance:** a fixture whose showcase imports a global reset (incl. `html`/`body` rules) and a portal component: the shell is unaffected, the showcase is styled by the user's CSS, the portal is not clipped and paints above the shell; per-showcase scheme switching (`colorScheme` prop, `data-scheme` backdrop); README heading links scroll to the heading; theme fonts load; a dark theme gives dark scrollbars and canvas.
  - **Contrast smoke test (D85):** computed WCAG AA contrast for the default themes' semantic pairs in the built output.
- **Depends on:** C11.

### C13 — Documentation

- **Goal:** user-facing and author-facing docs, consolidated and complete.
- **Requirements:**
  - Root README (what tsshowcase is, quick start, CLI reference).
  - Per-package READMEs completed.
  - Focus-management libraries and Shadow DOM (the Fluent/Tabster worked example, D95); stacking of slotted showcase content under the shell's z-indexed elements.
  - Module authoring guide (incl. the isolation model, D89), theme authoring guide (expanding "Building a theme that works with any module", D86), test-runner authoring guide; each points to the matching `@ofzza/tsshowcase-lint` preset (D88).
  - Interface changelog reviewed for the first `0.1.0` publish (D80).
- **Depends on:** C11 (can run in parallel with C12).

### C14 — Release

- **Goal:** v1 publishable (D31–D33).
- **Requirements:**
  - **Versioning (D80):** the first publish is `0.1.0`; the PR changeset check is switched on after it; PUBLISH.md documents the `0.x` major-changeset gotcha; decide whether `0.x` releases use a `next` dist-tag.
  - **CI extended:** Node 26 added to the `[22, 24]` matrix C1 already runs (D54); E2E (C12) in CI.
  - **Pack + install smoke test:** pack each package, install the tarballs into a scratch project, and run `check` and `build` there.
  - **Manual publish script** (no automated publish):
    - applies changesets versions;
    - runs the full CI suite and the smoke test;
    - publishes changed packages in dependency order (`-api` first, `-lint` after it), with scoped public access;
    - tags the release.
    - Supports a dry-run mode.
  - **Dogfood docs site (D41):** a GitHub Pages workflow that runs `tsshowcase build` on this repo. The structure already exists from C1 onwards.
- **Depends on:** C12, C13.

## C2 CLI duties → chunk (K10)

The C2 contract implies CLI-side behaviour beyond scanning and loading. Owners:

| Duty | Chunk |
| ---- | ----- |
| Read tracking for `RepoFiles` / `DirectoryInput` (D40) | C4 |
| Validating the analysis (unit dirs exist, inside their package and `unitRoot`, unique, known packages) and throwing a module error (D70) | C9 (shared helper, reused by C11) |
| Dropping units and `analyze` issues outside `scope` (D106; order: existence check → drop → remaining D70 checks) | C9 (shared helper, reused by C11) |
| Severity overrides with per-package `rules`, deriving `Issue.package` (D70) | C9 (shared helper, reused by C11) |
| Synthetic `module/error` issue when `verify` throws | C9 |
| Building the hierarchy from flat units + `unitRoot` (groups, nesting, group READMEs) | C11 |
| Defaulting units absent from `TestResults` | C11 |
| `SOURCE_DATE_EPOCH` for `generatedAt` | C11 |
| `revision` numbering and dropping out-of-order updates | C11 |
| Shutdown order (abort signals → `DevSession.close()` → `dispose()`) | C11 (and C9 for `dispose` after `check`/`init`) |
| Re-running `verify` in `dev` on read-tracked changes, re-analysis on `AnalysisResult.inputs` changes | C11 |
| Theme selection, duplicate names, scoping, CLI cache and atomic writes (D82) | C11 |

## Dependency edges

| Chunk | Depends on                                           |
| ----- | ---------------------------------------------------- |
| C1    | –                                                    |
| C2    | C1 (placement only)                                  |
| C3    | C2 (theme manifest; fills the `-api` catalogue, D28) |
| C4    | C2                                                   |
| C5    | C2, C3                                               |
| C6    | C2, C4 (workspace detection, scanner inputs; K9)    |
| C7    | C2                                                   |
| C8    | C2, C4 (fixtures)                                    |
| C9    | C4, C6, C8                                           |
| C10   | C2, C3, C5, C7, C8, C15                              |
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
| P3   | **C4, C5, C6, C7, C15** (approved one at a time, in any order, except **C4 before C6**) | Each depends only on C2/C3 (C6 also on C4's seam, K9). Planning C7 early also explores the Vitest Node API, the least-known piece; since C2 is approved, a finding there means backtracking to C2. |
| P4   | **C8**                                          | Needs C4's fixtures.                                                                                                                          |
| P5   | **C9** and **C10** (parallel)                   | C9 needs C4, C6 and C8; C10 needs C3, C5, C7, C8 and C15. The loadable module class lives in C8, so they don't depend on each other.          |
| P6   | **C11**                                         | Integrates C9 and C10.                                                                                                                        |
| P7   | **C12** and **C13** (parallel)                  | Both depend only on C11.                                                                                                                      |
| P8   | **C14**                                         | Final.                                                                                                                                        |

## Implementation order (stage 4)

`C1 → C2 → C4 → C6 → C8 → C9` gives a **usable `check` + `init` early**. The point (D29) is to **lock down the inputs** (scanning, analysis, the data model) against real code before they are shaped into a themed showcase app.
Then `C3 → C5 → C7 → C15 → C10 → C11 → C12 → C13 → C14`. C3's implementation also extends `-api` with the catalogue constants.

If implementation reveals a contract gap, we backtrack to the C2 plan, edit it (in place before the first publish; with an `-api` changeset after it, D80), and re-approve.
