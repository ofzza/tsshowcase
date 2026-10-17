# Super-plan — tsshowcase v1

> Status: **APPROVED** (after a final consistency review, see `01a-super-plan-review.md`). Decisions referenced as `D<n>` are in AGENTS.md §3.

## Packages (D14, D15, D26, D27, D43)

| Package | Role |
|---------|------|
| `@ofzza/tsshowcase-api` | Contracts only: TS types and interfaces, `IMPLEMENTED_API_VERSION`, interface changelog, small runtime helpers (e.g. issue constructors). Has no heavy deps. |
| `@ofzza/tsshowcase` | The CLI (binary `tsshowcase`): repo scanning, markdown pipeline, plugin loading, config, data assembly, `dev` file watching, and the `check` / `build` / `dev` / `init` actions. It doesn't bundle anything itself; modules build (D43). Depends on the built-in packages so they can be looked up by name (D9). |
| `@ofzza/tsshowcase-theme-default` | Default themes: built-in names `default-light` and `default-dark`. Also holds the reference documentation for the theme catalogue. |
| `@ofzza/tsshowcase-testrunner-vitest` | Prepackaged Vitest test runner (D3), with run and watch modes. Built-in name: `vitest`. |
| `@ofzza/tsshowcase-base-vite` | Reusable Vite pipeline helper (D43). Not a loadable module. Handles: data embedding as a virtual module, theme CSS inclusion and switcher setup, `base: './'`, output dir, dev server, HMR and the live update channel. Modules call it with their framework plugins and app entry. |
| `@ofzza/tsshowcase-base-react` | Shared React base (D20, D26, D48). Layout-agnostic and **not itself a loadable module**. Two halves:<br>• **check/init (Node, no bundler):** static entry analysis, base rules and the `headless` preset, scaffold templates;<br>• **build/dev:** a **React wrapper around `-base-vite`** (plugin-react, React deduped from the showcased repo, tsconfig paths) plus the runtime building blocks (data and live-channel access, README HTML renderer, showcase embed, test-results view, theme switcher, hash router).<br>Exposes a **base module class** with analyze / verify / scaffold / runTests / build / dev already implemented. Layout modules extend it and supply only their app entry and layout. Depends on `-base-vite`. |
| `@ofzza/tsshowcase-module-react-default` | The v1 rendering module (D22). Built-in name: `react-default`. Contains **layout only**: app shell, navigation tree (repo → packages → units), route scheme, and the home/package/unit pages composed from the base building blocks. |

## Chunks

Each chunk lists **Goal**, **Requirements** and **Depends on**. Each chunk delivers its own docs, its tests and the fixtures it needs (D10 conventions; docs ship with code).

### C1 — Monorepo foundation
- **Goal:** an empty but fully wired workspace.
- **Requirements:**
  - npm workspaces with the package skeletons listed above.
  - TS (ESM, Node ≥ 20, D17), build via `tsc` project references.
  - Prettier (160), ESLint, Vitest workspace config, `npm run ci` aggregate script.
  - A `fixtures/` directory convention. Fixtures are **isolated from the workspace**: excluded from `workspaces`, from root lint/test/type-check, and from dependency hoisting (each fixture has its own install where it needs `react`/`vitest`). Otherwise "resolve React from the showcased repo" can't be tested.
  - **Basic CI (D32):** a GitHub Actions workflow that runs lint, type-check, test and build on push/PR, so every chunk is checked.
  - changesets set up for independent versioning (D31).
  - The D10 conventions documented in AGENTS.md.
  - **Dogfooding (D41):** the package skeletons follow the unit convention (a dir per exported unit, README, tests, showcases per D37).
- **Depends on:** nothing.

### C2 — Plugin contracts (`-api`)  ★ critical path, designed collaboratively in detail before any code (assignment requirement)
- **Goal:** the complete, documented v1 contracts.
- **Requirements:**
  - **Input data model:**
    - repo info (root, VCS host, branch);
    - packages[] (D16, D19), each with: package info (sourced from package.json in v1, but source-agnostic so other repo types can come later), licence (D12), README raw and HTML (D23), unit hierarchy;
    - how the repo root README/licence relate to package READMEs/licences, including the single-package collapse (no duplicated README on home and package page);
    - per unit (a directory exporting 1+ symbols; nesting allowed, D36, so a unit can be both a page and a parent of sub-units): kind, exported symbols, README raw/HTML, tests (hierarchy plus outcomes), **optional** showcase entry path (D37);
    - extensible: additive only (D13).
  - **Rendering module interface** (default-exported class, D9 loading):
    - `apiVersion` (semver range, D13);
    - `analyze(repo)` → exported units/context (D5);
    - `verify(dir, context)` → `{ ignored } | { issues[] }` (D2), receiving the file list plus lazy, cached `read(file)` accessors (D40);
    - `runTests(ctx)` (D3, D34, D42): the module calls `ctx.loadTestRunner(ref?)` (an explicit ref from config or the module default, otherwise auto-detection), runs or watches, maps the raw results to units, and returns them. In dev, test updates are pushed via `ctx.onTestUpdate` (D18);
    - `resolveRoute(target, heading?)` (D35): the in-app URL for home, a package or a unit;
    - `build(ctx)` and `dev(ctx)` (D43): the module builds the static app, or runs a dev server. `ctx` carries the assembled data, the resolved theme CSS files, the output dir and (in dev) `onDataUpdate` / `onTestUpdate` subscriptions;
    - `scaffold(unit)` for `init` (D16);
    - module-specific options schema (D6);
    - room for a playground (D25).
  - **Inclusion rules for any module's output** (D43): how the data is embedded, how theme CSS is included and scoped, how the switcher attribute works. Mandatory even for modules that don't use `-base-vite`.
  - **Issue model:** severity, message, file, rule id.
  - **Test-runner interface:** `apiVersion`, `detect(repo)`, `run(options)`, `watch(options, onUpdate)`; a raw result hierarchy (file → suite → test, with status, duration and error); never throws on failing tests (D8, D42).
  - **Theme contract:** a theme manifest (`apiVersion`, name(s), CSS entry files, light/dark flags, D7).
  - **Theme catalogue constants (D28):** every documented CSS class name and CSS variable name, exported as typed constants that modules import. The *values* and the *design* of the hierarchy come from C3; C2 provides the slot and the export mechanism, and C3's plan fills it in (it extends `-api`).
  - **Config file schema** (D6).
  - **Data handed to `build`/`dev`:** its shape, plus the dev update protocol (data changes and test updates, D18).
  - **Repo-level verification:** issues not tied to one directory (e.g. an unresolvable entry, an export whose source is shared with other exports). Covered by a repo-level hook or by `analyze` returning issues.
  - **Output tolerance:** the CLI tolerates unknown fields in module outputs (analysis results, issues, test results) as well as in inputs (D13).
  - Interface changelog started at 1.0.0.
- **Depends on:** C1 (only for where the code lives; planning is independent).
- **Note:** the C2 plan stays *open* until the C3 plan is approved, because C3 fills in the catalogue constants in `-api` (D28). Implementing C3 modifies the `-api` package.

### C3 — Theme system + default themes
- **Goal:** design the CSS variable hierarchy and class catalogue (the theme ⇄ module contract; published as constants in `-api`, D28), plus the light and dark themes.
- **Requirements:**
  - **Layered variables:**
    - base tokens (primary/secondary colours, base spacing, base font size, radius, …);
    - → derived scales (palette ramps, spacing scale, type scale) computed from the base tokens (`calc()`, `color-mix()`);
    - → semantic tokens (surface, text, link, border, code background, success/failure, …);
    - → component tokens.
  - Overriding any layer cascades to the layers below it.
  - Documented, `tss-`-prefixed class catalogue, including region classes (D44): layout shell, nav, headings, paragraphs, links, lists, tables, code (inline and block), blockquote, showcase frame and error, test-result list and status, theme switcher.
  - Covers the markdown HTML output (C5).
  - Runtime theme switching with a `prefers-color-scheme` default (D7). Persists the user's choice and avoids a flash of the wrong theme on load.
  - **Theme scoping:** how several themes coexist in one bundle (e.g. `[data-theme=…]` scoping), and how a custom theme can extend a built-in one.
  - **Three-tier scope (D44):**
    - variables are inherited everywhere, including showcases;
    - catalogue classes work on any element;
    - opt-in region classes (e.g. `tss-content`) theme bare elements inside them;
    - no bare-element styling outside regions, so user components are never restyled accidentally;
    - shell CSS goes in a cascade `@layer`, scoped to shell classes.
  - Prefix `tss-` / `--tss-` (D45).
  - **Enforcement tooling (D46):** one policy, enforced by stylelint (CSS) and ESLint (inline styles / CSS-in-JS), for tsshowcase's own module and theme packages only:
    - allowed: `var(--tss-…)`, `calc()`/`color-mix()` of them, structural keywords;
    - banned: literal design values;
    - bare element selectors are banned outside region classes (D44);
    - escape hatch: a line-level disable comment with a required reason.
  - Docs listing every variable and class.
- **Depends on:** C2 (theme manifest shape only). The catalogue design itself is independent.

### C4 — Repository scanning (CLI core)
- **Goal:** turn a repo on disk into the repo / package / directory layer of the C2 input model. Units come from the module's `analyze` (C8), tests from the runner (C7), and HTML from C5.
- **Requirements:**
  - `.gitignore`-aware recursive walk:
    - nested `.gitignore` files and `.git/info/exclude`;
    - always ignored: `.git`, `node_modules` and the configured output directory;
    - defined behaviour for non-git directories.
  - Lazy file content accessors.
  - Workspace detection (npm, yarn and pnpm workspaces); single package = one package (D16).
  - package.json info, licence detection (D12), root and package READMEs.
  - Git info: remote → host, default branch (D24), with a deterministic fallback for shallow clones, detached HEAD and a missing `origin/HEAD`.
  - Fixtures: a single-package repo, a workspace repo, and broken variants.
- **Depends on:** C2.

### C5 — Markdown pipeline (CLI core)
- **Goal:** markdown → themed HTML (D23).
- **Requirements:**
  - unified/remark/rehype with GFM and heading slugs.
  - Emits only C3 catalogue classes.
  - **Link rewriting:**
    - unit README → in-app URL via the module's `resolveRoute` (D35);
    - repo file → host URL template (GitHub/GitLab/Bitbucket, overridable, D24);
    - external links left untouched;
    - heading anchors (`#slug`, in-page or cross-README) rewritten via `resolveRoute(target, heading)` (D35), since bare `#slug` collides with hash routing.
  - Syntax highlighting for code blocks (D39), emitting only themable token classes, with no inline colours.
- **Depends on:** C2, C3 (class names).

### C6 — Plugin loading + configuration (CLI core)
- **Goal:** resolve and validate modules, themes and runners (runners on the module's behalf, D34), and the effective config.
- **Requirements:**
  - Reference resolution order: built-in name → npm package → path (D9).
  - TS loading via jiti.
  - Default-export class instantiation.
  - `apiVersion` semver check with clear errors (D13).
  - `ctx.loadTestRunner(ref?)` helper: resolves an explicit ref, or auto-detects among candidate runners (D42).
  - Config file discovery (`tsshowcase.config.ts|js|json`), validated against the C2 schema, merged with CLI flags (D6).
  - Validation of module options.
- **Depends on:** C2.

### C7 — Vitest test runner (`-testrunner-vitest`)
- **Goal:** the D3 prepackaged runner.
- **Requirements:**
  - Runs the showcased repo's Vitest via the Vitest Node API, using the repo's own vitest install and config.
  - Returns the **raw** C2 test hierarchy (file → describe → test, with status, duration and error message). Mapping results to units is the module's job (D42), not the runner's.
  - Watch mode that streams updates (D18).
  - `detect(repo)`: vitest in deps, `vitest.config.*`, a `test` block in `vite.config.*`, workspace/projects config (D42).
  - A failing test never throws (D8).
- **Depends on:** C2.

### C8 — React base: Node side (`-base-react`)
- **Goal:** everything `check` and `init` need for React/TS repos. Also the first version of the React **base module class** (Node-side methods: analyze / verify / scaffold), **plus the loadable `-module-react-default` class** extending it, so that C9 can run `--module react-default` before C10 exists. No bundler involvement (D48).
- **Requirements:**
  - **Static entry analysis** with the TS compiler API (D5):
    - package.json `exports`/`main`/`types` → entry → follows `export *` and named re-exports → source directories;
    - maps built entries (`dist/…`) back to source (e.g. the `source` export condition, tsconfig `rootDir`/`outDir`, or a config override), since `dist` may not exist;
    - uses the showcased repo's own TypeScript if it has one;
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
- **Depends on:** C2, C4 (fixtures).

### C9 — CLI shell + `check` + `init`
- **Goal:** the first usable vertical slice.
- **Requirements:**
  - CLI framework, global flags (`--module`, `--theme`, `--config`, `--cwd`).
  - **`check`:**
    - tree report of every non-ignored directory, with per-directory issues (human-readable, coloured);
    - `--format json`;
    - non-zero exit code on errors (D16).
  - **`init <unit>`** calls the module's `scaffold` (and reports clearly when a module lacks it, per D13 feature detection).
  - **Self-check (D41):** `tsshowcase check` runs on this repo in CI.
- **Depends on:** C4, C6, C8.

### C10 — React runtime building blocks + default module (`-base-react` runtime, `-module-react-default`)
- **Goal:** the generated SPA. The base module class gains `build`/`dev`, implemented as a React wrapper around `-base-vite` (C15), plus the runtime building blocks. `-module-react-default` adds only its app entry and layout (D48).
- **Requirements:**
  - **React input to `-base-vite` (C15)** and `build`/`dev` implementations that call it:
    - `@vitejs/plugin-react`;
    - react/react-dom **resolved and deduped from the showcased repo** (pitfall: duplicate React). The dedupe also covers module runtime files living in the CLI's `node_modules`, and the module's packages must not pull in their own React copy through peer auto-install;
    - respects the repo's tsconfig paths;
  - **Base class `runTests`** (D42): `ctx.loadTestRunner` → run / watch → map raw results to units (by unit directory, incl. nested units, D36) → return / push via `ctx.onTestUpdate`.
  - **App:**
    - hash router;
    - nav tree (repo → packages → units, D19; nested units are both pages and parents, D36);
    - home page = repo README; package pages with README and licence; unit pages with README HTML, the showcase (if any, D37) in Suspense and an error boundary, and test results;
    - `resolveRoute` implementation (D35);
    - theme switcher UI component (persistence and no-flash init live in C15).
    - showcases of the `-base-react` components themselves (dogfooding, D41), using mock data.
  - **Only C3 classes and variables are used.** Passes the C3 stylelint and ESLint rules.
  - Consumes the embedded data and the dev live channel provided by `-base-vite` (C15).
- **Depends on:** C2 (data, test-result and live-channel shapes), C3, C5 (HTML shape), C7 (runner, for `runTests`), C8 (base and the module class), C15 (Vite helper).

### C11 — CLI `build` + `dev`
- **Goal:** orchestration. The CLI assembles data and calls the module's `build`/`dev` (D43); it doesn't bundle anything itself.
- **Requirements:**
  - **Data assembly:** C4 scan → module `analyze` → module `runTests` (which uses C7) → C5 HTML (links via `resolveRoute`) → `module.build(ctx)` / `module.dev(ctx)`.
  - Theme resolution (several themes, D7); the theme files are passed in `ctx`.
  - **`build`:**
    - output to `--out` (default `./showcase`);
    - **output guard:** refuse to write into a non-empty directory that tsshowcase didn't create (e.g. stdblox already has a `showcase/` source dir);
    - `--fail-on-test-failure` (D8); `--no-tests` (optional). The chunk plan decides whether "tests unavailable" (D42) also fails in strict mode, and whether test results are cached.
  - **`dev`:** the CLI watches files. README changes are re-rendered, and adding or removing a unit or file triggers re-analysis. The new data is pushed to the module's running dev server (`ctx` update subscription). Test updates come from the module's watch mode (D18, D42).
- **Depends on:** C5, C6, C7, C9, C10.

### C15 — Vite pipeline helper (`-base-vite`)
- **Goal:** the reusable Vite build/dev pipeline that modules call (D43). Numbered C15 to avoid renumbering existing chunks.
- **Requirements:**
  - API: `build({ data, themes, outDir, plugins, entry, resolve… })` and `dev({ …, onDataUpdate, onTestUpdate })`.
  - Data embedding as a virtual module; theme CSS inclusion and scoping, plus the switcher setup (`[data-theme]`, `prefers-color-scheme` default, persisted choice, a no-flash init script in the HTML head), per the C2 inclusion rules and the C3 theme scoping.
  - `base: './'`; output dir; `server.fs.allow` for module runtime files outside the repo root.
  - Dev server, HMR, and pushing data and test updates to the browser (D18).
  - Framework-agnostic: no React knowledge.
- **Depends on:** C2, C3.

### C12 — End-to-end acceptance
- **Goal:** proof that v1 works as a whole.
- **Requirements:**
  - E2E matrix over the fixtures (D11): check / init / build / dev smoke, for single-package and workspace repos.
  - Built output opened in headless Chrome: navigation, showcase rendering, test results, theme switching.
- **Depends on:** C11.

### C13 — Documentation
- **Goal:** user-facing and author-facing docs, consolidated and complete.
- **Requirements:**
  - Root README (what tsshowcase is, quick start, CLI reference).
  - Per-package READMEs completed.
  - Module authoring guide, theme authoring guide, test-runner authoring guide.
  - Interface changelog reviewed for 1.0.0.
- **Depends on:** C11 (can run in parallel with C12).

### C14 — Release
- **Goal:** v1 publishable (D31–D33).
- **Requirements:**
  - **CI extended:** a Node version matrix (20/22/24); E2E (C12) in CI.
  - **Pack + install smoke test:** pack each package, install the tarballs into a scratch project, and run `check` and `build` there.
  - **Manual publish script** (no automated publish):
    - applies changesets versions;
    - runs the full CI suite and the smoke test;
    - publishes changed packages in dependency order (`-api` first), with scoped public access;
    - tags the release.
    - Supports a dry-run mode.
  - **Dogfood docs site (D41):** a GitHub Pages workflow that runs `tsshowcase build` on this repo. The structure already exists from C1 onwards.
- **Depends on:** C12, C13.

## Dependency edges

| Chunk | Depends on |
|-------|-----------|
| C1 | – |
| C2 | C1 (placement only) |
| C3 | C2 (theme manifest; fills the `-api` catalogue, D28) |
| C4 | C2 |
| C5 | C2, C3 |
| C6 | C2 |
| C7 | C2 |
| C8 | C2, C4 (fixtures) |
| C9 | C4, C6, C8 |
| C10 | C2, C3, C5, C7, C8, C15 |
| C11 | C5, C6, C7, C9, C10 |
| C12 | C11 |
| C13 | C11 |
| C14 | C12, C13 |
| C15 | C2, C3 |

## Planning order (stage 3)

Plans are approved one at a time. "Parallel" means a group's members only depend on chunks already planned, so they could be planned in any order or at the same time.

| Wave | Chunks | Why |
|------|--------|-----|
| P1 | **C1** | Small; fixes the layout everything else refers to. |
| P2 | **C2**, then **C3** | C2 is the critical, collaborative contract. C3 fills C2's catalogue, so the C2 plan stays open until C3 is approved. |
| P3 | **C4, C5, C6, C7, C15** (any order or parallel) | Each depends only on C2/C3. Planning C7 early also prototypes the Vitest Node API, the least-known piece, before the contracts are locked in. |
| P4 | **C8** | Needs C4's fixtures. |
| P5 | **C9** and **C10** (parallel) | C9 needs C4, C6 and C8; C10 needs C3, C5, C7, C8 and C15. The loadable module class lives in C8, so they don't depend on each other. |
| P6 | **C11** | Integrates C9 and C10. |
| P7 | **C12** and **C13** (parallel) | Both depend only on C11. |
| P8 | **C14** | Final. |

## Implementation order (stage 4)

`C1 → C2 → C4 → C6 → C8 → C9` gives a **usable `check` + `init` early**. The point (D29) is to **lock down the inputs** (scanning, analysis, the data model) against real code before they are shaped into a themed showcase app.
Then `C3 → C5 → C7 → C15 → C10 → C11 → C12 → C13 → C14`. C3's implementation also extends `-api` with the catalogue constants.

If implementation reveals a contract gap, we backtrack to the C2 plan, bump the interface changelog, and re-approve.
