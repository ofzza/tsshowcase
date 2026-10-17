# tsshowcase — agent notes

This file is the **single source of truth for project state**. Keep it current. Every Q&A answer, decision, approval and backtrack is recorded here. That way work can stop and resume at any point.

The original assignment is in [REQUIREMENT.md](./REQUIREMENT.md) (verbatim; never edit it).

---

## 1. Assignment (summary)

Extract the showcase/enforcement idea from `~/projects/ofzza/blox-fluentui-react/stdblox-react` into a standalone CLI utility.

**CLI actions**

- **check**: scans a repository and enforces that every publicly exported component, service or utility:
  - lives in its own directory;
  - has a `README.md`;
  - has a test suite;
  - has a showcase page (later possibly an interactive playground).
  - *Changed by D37/D38:* README is always required; tests are required except for type-only units; whether a showcase is required depends on the unit kind and is decided by the module.

  It outputs a report of the repository structure (all directories not ignored by `.gitignore`) with the issues found for each directory.
- **v1 also adds** `dev` and `init` (D16).
- **build**: builds a static, deploy-anywhere (e.g. GitHub Pages) showcase app from those resources. Output goes to a path argument, defaulting to `./showcase`. It uses the selected rendering module and theme.

**Modularity**

1. **Rendering modules** implement a clearly defined interface.
   - A module's default export is a class implementing the interface. The CLI imports it at runtime.
   - The CLI provides these inputs:
     - package info (`package.json` first; other repo types later);
     - the licence file contents;
     - the root `README.md`;
     - a component hierarchy, with for each unit: its README contents, its tests and their status, and the path to its showcase page;
       - *changed by D3/D42:* the **module** produces the tests and their status, using a prepackaged runner; the CLI does not;
     - room to extend later.
   - Modules also expose a **verification** method. It receives every non-ignored subdirectory together with its file list and file contents (*changed by D40:* lazy, cached file accessors rather than eager contents). It decides whether to ignore the directory and, if not, whether the directory meets the module's requirements.
   - The CLI repo ships built-in modules (a React module for TS/React first). The user can pick a built-in module or point to a custom one by path.
   - The interface must be designed **collaboratively and in detail before any code is written**.
2. **Theming**
   - A documented, hierarchical system of CSS variables, where derived variables extend from base ones. For example, override only primary/secondary colours, or the whole palette, or only the base spacing, or the whole spacing scale.
   - Documented CSS class names that modules must use (links, titles, paragraphs, code blocks, …).
   - **Rule:** no styling may ever be implemented unless it uses an exposed variable or class, or is derived directly from an exposed variable.
   - Built-in themes ship in the repo (default light and default dark first). The user can pick a built-in theme or point to a custom one by path.

## 2. Process & current state

Stages (each one is interactive and needs explicit user approval; backtracking is allowed at any time):

| # | Stage | Status |
|---|-------|--------|
| 0 | Bootstrap: REQUIREMENT.md, AGENTS.md, CLAUDE.md, initial commit | ✅ done |
| 1 | Requirements Q&A: until the requirements are clear | ✅ done (D1–D25). D26–D48 were decided during super-planning; details continue in the chunk plans |
| 2 | Super-plan: partition the work into chunks, order them, mark which can be planned in parallel and which sequentially; iterate until approved | ✅ **approved**: `.plans/01-super-plan.md`, with the review triage and final consistency review in `.plans/01a-super-plan-review.md` |
| 3 | Chunk plans: one at a time, in super-plan order; iterate and approve each | 🔜 next: **C1 (Monorepo foundation)**, wave P1 |
| 4 | Implementation: one chunk at a time; explain each and wait for review before the next | ⏳ |

Rules:
- After every completed stage step (a plan is approved, or a chunk is implemented), save the plan to `.plans/` and **commit all changes**.
- Plan file naming: `.plans/00-bootstrap.md`, `.plans/01-super-plan.md`, `.plans/chunk-NN-<slug>.md`.
- Commit messages carry **no AI attribution** (no Co-Authored-By, no session links). This is the user's global rule.
- Never write credentials anywhere.

## 3. Decisions log

| ID | Decision | Notes |
|----|----------|-------|
| D1 | The repo is `~/projects/ofzza/tsshowcase` | It already existed as an empty git repo; the branch is `master` |
| D2 | `verify` **returns a list of issues** instead of throwing | Return shape is roughly `{ ignored } \| { issues: [{ severity: error\|warning, message, file? }] }`. The report can then show every issue per directory |
| D3 | **The rendering module is responsible for running tests** and returns a test hierarchy plus outcomes, in a shape the interface defines | The CLI repo provides **prepackaged test runners** (Vitest-compatible first) that modules can reuse |
| D4 | ~~**The CLI owns Vite**; modules contribute config~~ **Superseded by D43** | A module provides its framework plugins, app entry and runtime deps (e.g. a `getViteConfig(context)` method). An optional module `build()` override may be added later as an escape hatch. The showcased repo is never tied to Vite. Framework runtime packages (react, react-dom, …) **must resolve from the showcased repo**, while bundler plugins ship with the CLI/module |
| D5 | "Publicly exported" units are found by **static entry analysis** | The module analyses the package entry (`package.json` `exports`, then `index.ts`) with the TS compiler API and follows re-exports back to their source directories. This replaces stdblox's string match. **Consequence:** verification needs a repo-wide analysis step (e.g. `analyze(repo)`, which produces exported units / context) *before* per-directory `verify(dir, context)` |
| D6 | Configuration is an **optional config file plus CLI flags** | `tsshowcase.config.(ts\|js\|json)` holds the module, theme(s), module-specific options and output. CLI flags override the file |
| D7 | Themes: **several per build, with a runtime switcher** | The default follows `prefers-color-scheme`. The theme system defines a switcher class/slot that modules render |
| D8 | **Failing tests never block `build`** | Failures are shown per unit in the showcase. A `--fail-on-test-failure` flag gives strict mode (CI) |
| D9 | **How module/theme references resolve, in order:** (1) the name of a module/theme bundled with the CLI, (2) an installed npm package name, (3) a filesystem path | JS and TS are both allowed. TS is loaded at runtime with a loader (e.g. jiti), so custom module authors need no build step |
| D10 | **Adopt the stdblox code conventions** | Prettier `printWidth: 160`; British spelling; file header comments; JSDoc everywhere; `_` prefix for internals; `// #region` blocks; a purpose comment opening every `describe` |
| D11 | **Validation uses in-repo fixtures only** | Do not touch stdblox-react for now |
| D12 | **Licence file:** found case-insensitively as `LICEN[CS]E` with an optional `.md`/`.txt` extension, for each package in workspaces | Confirmed |
| D13 | **Contract versioning uses a semver range declared in code** | Every module, theme and test runner declares `static readonly apiVersion = '^1.2'`.<br>• **The interface has its own version**, independent of the CLI package version, plus an **interface changelog** ("added in 1.3: …"). It is exported as `IMPLEMENTED_API_VERSION`.<br>• The CLI checks `semver.satisfies(IMPLEMENTED_API_VERSION, apiVersion)` at load time and fails with a clear message.<br>• **Within a major version, changes are additive only.** Methods added after 1.0 are optional and feature-detected; modules ignore fields they don't know.<br>• The types package is versioned in step with the interface.<br>• **Rejected:** requiring a `package.json` with a peerDependency on the interface package for path-loaded modules |
| D14 | **This repo is an npm workspaces monorepo** | Separate packages (full list: super-plan package table): `-api`, the CLI, themes, the test runner, `-base-vite`, `-base-react` and the React module. The CLI depends on the built-in packages so that lookup by name works (D9) |
| D15 | **Naming:** `@ofzza/tsshowcase` (the CLI, binary `tsshowcase`), with siblings `@ofzza/tsshowcase-<kind>-<name>` | See D27 for the full scheme |
| D16 | **v1 scope adds:** a `dev` action (watch + HMR; module-owned via D43); an `init`/scaffold action (generate the stubs a module requires); **workspace (monorepo) showcased repos**; a machine-readable check report (`--format json`, non-zero exit on errors) | **Interface consequences:**<br>• the module needs a scaffold method;<br>• the input model must be multi-package from day one (a single-package repo = one package);<br>• `dev` needs a test strategy |
| D17 | **Node >= 20, ESM only** | |
| D18 | **In `dev`, tests run in watch mode** | Results are pushed live to the open showcase through the module's dev server (D42, D43), so the test-runner interface must support watch and streamed updates |
| D19 | **Showcase hierarchy for workspaces:** repo README (home) → each package (its README and licence) → that package's units | The package level collapses for a single-package repo |
| D20 | **Layout variants are separate rendering modules** (e.g. a React per-page module vs a React single-page module) | They share a common React base (resolved by D26/D48: the `-base-react` package with a base module class) |
| D21 | **React verification:** base rules by default, opt-in rule presets | Base rules: own directory, README, test, showcase, exported from the entry (test and showcase requirements refined by D37/D38). The headless pattern (`useX`/`renderX`/`X`, from stdblox) is an opt-in preset named `headless` (D47) |
| D22 | **Only the React default module (`react-default`, D27) ships in v1** | The shared base (D26) makes further React modules cheap to add |
| D23 | **The CLI pre-renders markdown to HTML** | Output carries the documented theme classes, heading ids and rewritten links. Modules receive both the raw markdown and the HTML |
| D24 | **How repo-file links are rewritten:** the host (GitHub/GitLab/Bitbucket) is detected from `package.json` `repository` and the default branch from git; a configurable URL template overrides both | Links to unit READMEs become showcase routes |
| D25 | **Interactive playground is out of v1 scope** | The interface should leave room for it later |
| D26 | **The React base ships as a separate package from v1:** `@ofzza/tsshowcase-base-react` | **Base:** layout-agnostic parts (analysis, rules, scaffolding, test-result mapping, the React wrapper around `-base-vite` per D48) and runtime building blocks. It is not a loadable module.<br>**Default module:** layout only (shell, nav, routes, page composition). The user chose this over the recommended internal split |
| D27 | **Naming scheme `@ofzza/tsshowcase-<kind>-<name>` (kinds: `module`, `base`, `theme`, `testrunner`; the test-runner kind was renamed from `runner` to be explicit):**<br>• `-module-react-default` (built-in name `react-default`; renames "react-pages");<br>• `-theme-default` (`default-light`, `default-dark`);<br>• `-testrunner-vitest` (`vitest`);<br>• `-base-react`;<br>• `-api` | "single-page vs pages" was only an example of why several modules might exist, not a v1 requirement |
| D28 | **The theme catalogue lives in `-api` as importable typed constants** | Every class name and CSS variable name. Modules import the constants and never hardcode strings. C3 designs the hierarchy and implements the default themes |
| D29 | **Implementation order builds the "inputs" first** (`check`/`init` slice: C1→C2→C4→C6→C8→C9) | Lock down the inputs (scanning, analysis, data model) before moulding them into a themed showcase app |
| D30 | **The former C12 is split** into C12 (E2E acceptance), C13 (docs) and C14 (release) | Release scope settled in D33 |
| D31 | **Packages are versioned independently, with changesets** | Clarifies D13: **the `-api` package version *is* the interface version** (`IMPLEMENTED_API_VERSION` is derived from it), and changesets bumps it like any other package. "Independent" means the other packages' versions don't follow it |
| D32 | **Basic CI from C1** (lint, type-check, test, build on push/PR) | C14 extends it |
| D33 | **C14 scope:** CI extensions (Node matrix, E2E), pack-and-install smoke test, a **manual publish shell script** (dependency order, dry-run; no automated publish workflow for now), and a dogfood docs site on GitHub Pages | |
| D34 | **Test ownership: the module's `runTests()` calls a runner library.** The runner is chosen through a **module option** (`moduleOptions.testRunner`, e.g. `'vitest'`; auto-detected if unset, D42) | The CLI resolves runner references (built-in name → npm → path, D9) on the module's behalf, but never calls a runner itself. Refines D3 |
| D35 | **Routes: the module exposes `resolveRoute(target, heading?)`** | `target` is home, a package or a unit. The CLI's markdown renderer (D23) calls it to rewrite README file links into final in-app URLs. Links to non-README repo files go to the VCS host (D24) |
| D36 | **A unit is a directory exporting one or more public symbols. Nesting is allowed** (a unit may contain sub-units) | Intermediate non-unit directories become menu levels |
| D37 | **Requirement change: README and tests are mandatory for every unit; showcases are mandatory per kind, as the module decides** | `react-default`: required for components and hooks; optional for functions, classes and constants; not applicable to type-only units. Severity can be overridden via rule IDs (D21). The showcase path is optional in the data model, and layouts must handle its absence. Pure-TS libraries therefore need no React unless they opt into showcases |
| D38 | **Type-only units require only a README** | Tests are optional; type tests (`expectTypeOf`, `*.test-d.ts`) are shown if present. Exception to D37's "tests mandatory" |
| D39 | **Syntax highlighting is kept, class-based only** | Done at build time in the CLI markdown pipeline (e.g. Shiki with a CSS-variables theme, or Prism/highlight.js classes). Token colours are catalogue variables; no inline colours |
| D40 | **`verify` receives the file list plus lazy, cached `read(file)` accessors** instead of eager contents | Requirement change: semantically equivalent, but only files the module needs are read, and the CLI can track reads for `dev` invalidation |
| D41 | **This repo dogfoods its own convention from C1** | Every exported unit in this repo's packages has its own dir, README and tests, with showcases per D37 (the `-base-react` components need them). `tsshowcase check` runs on this repo in CI once C9 lands. C14 only adds the GitHub Pages deploy |
| D42 | **Test flow and runner loading** (refines D3/D34):<br>• the CLI calls `module.runTests(ctx)`;<br>• the module calls `ctx.loadTestRunner(ref?)` and gets back a resolved, version-checked runner;<br>• the runner's `run`/`watch` drive the **showcased repo's own** test tool and return a raw hierarchy (file → suite → test);<br>• the **module maps results to units** and returns them;<br>• in `dev`, `watch` updates go module → `ctx.onTestUpdate` → CLI → the module's dev server (`onTestUpdate` subscription) → browser | **Auto-detection:** runners implement `detect(repo)`.<br>• An explicit `ref` (config `moduleOptions.testRunner`, or a module default) wins.<br>• Otherwise the CLI calls `detect` on each candidate runner (built-ins plus extra candidates from the config). Exactly one match: use it and log it. Several: error with a hint. None: tests are unavailable (warning).<br>• Modules may restrict the candidates.<br>• The runner never throws on failing tests (D8). If the runner itself fails, the affected units show "tests unavailable" |
| D43 | **Hybrid build ownership (supersedes D4):** the contract says the **module owns `build(ctx)` and `dev(ctx)`**. The CLI assembles the data (packages, units, README HTML, tests), resolves the themes and the output dir, watches files and re-analyses in `dev`, and pushes updates to the module (e.g. `ctx.onDataUpdate`). | The **Vite pipeline ships as a reusable helper package, `@ofzza/tsshowcase-base-vite`**. It handles data embedding (virtual module), theme CSS inclusion and switcher setup, `base: './'`, the output dir, the dev server, HMR and the live channel. `react-default` uses it by passing its plugins and entry. Modules for other frameworks may bring their own toolchain, but must honour the data and theme inclusion rules defined in `-api`. Still applies: framework runtime packages resolve from the showcased repo (handled in `-base-react`). History: Claude first recommended module-owned, then CLI-owned (D4) after discussion, and finally the hybrid |
| D44 | **Three-tier theming scope:**<br>(1) **variables** are defined on the app root (`[data-theme=…]`, plus `color-scheme`) and inherited everywhere, including user showcases, so they follow light/dark automatically;<br>(2) **catalogue classes** (`tss-title`, `tss-link`, …) can be used on any element, including inside showcases, and look the same as in the shell;<br>(3) **opt-in region classes** (e.g. `tss-content`) theme *bare* elements (`h1`, `p`, `a`, `table`, `code`, …) inside them. README HTML is wrapped in a region, and showcases may use regions too | **Bare elements outside a region are never styled**, so user components look exactly as in their own app. Enforced by the C3 stylelint rule (bare element selectors only inside a region class). Shell CSS sits in a cascade `@layer` and is scoped to shell classes, to limit user CSS leaking in. Showcases may import class constants from `-api` (as a devDep) or use plain strings |
| D45 | **Prefix:** classes `tss-…`, CSS variables `--tss-…` | |
| D46 | **One styling policy, enforced in both CSS (stylelint) and inline styles/CSS-in-JS (ESLint).** Allowed: `var(--tss-…)` references, `calc()`/`color-mix()` built from them, and structural keywords (`display: flex`, `text-align: center`, `position`, `overflow`, `0`, `100%`, `auto`, `inherit`, …). Banned: literal design values (colours, lengths, font sizes, radii, shadows, durations, …) | Applies **only to tsshowcase's own module and theme packages**, never to user showcases. Escape hatch for genuinely dynamic values: a line-level disable comment that requires a written reason. This replaces the earlier idea of banning inline `style={}` entirely |
| D47 | **The opt-in React rule preset for the logic/presentation split is named `headless`** | `useX` holds the logic and prop getters, `renderX` is a pure presentation function, and `X` only wires them together. Config: `presets: ['headless']`. Not named after stdblox |
| D48 | **`-base-react` builds on `-base-vite`** | Its build/dev half is a React wrapper around `-base-vite` (plugin-react, React dedupe, tsconfig paths, runtime building blocks). Its check/init half (analysis, rules, scaffolding) has no bundler involvement. It exposes a **base module class** with every contract method implemented; layout modules (e.g. `react-default`) extend it and supply only their app entry and layout |

## 4. Open questions

- Detailed interface, theme and runner design happens in the respective chunk plans.

## 5. Prior art — stdblox-react findings

- **Units:** a directory with `index.ts` under `src/{components,hooks,utilities}/` (nesting allowed). Each unit has `README.md`, `*.test.ts(x)` and `*.showcase.tsx`. The showcase file default-exports a prop-less component.
- **Enforcement** is a Vitest test (`src/index.test.ts`) that uses `fs` walking plus `import.meta.glob`. Its rules are hardcoded: the unit roots, component file naming, and a brittle exact-string `export * from` match in `src/index.ts`. A second test renders every showcase as a smoke test.
- **Showcase app** (`showcase/`) is a Vite app:
  - It discovers content with `import.meta.glob` (READMEs eager as `?raw`, showcases lazy).
  - Markdown is rendered with `react-markdown`, `remark-gfm` and `rehype-slug`.
  - Routing is a home-made hash router.
  - `base: './'` keeps it working under any GitHub Pages subpath.
  - Each showcase has its own error boundary plus `Suspense`.
  - The menu tree mirrors the directory structure.
  - README links are rewritten: a unit README becomes a route, any other repo file becomes a GitHub blob URL.
- **Styling:** about 150 lines of CSS with six variables and a `prefers-color-scheme` dark variant. Test results are not surfaced anywhere.
- **Coupling to remove** when generalising:
  - hardcoded relative globs and paths (`../../src`, `../../package.json`);
  - showcase deps (react-markdown, …) living in the library's devDependencies;
  - GitHub-only link rewriting with the `master` branch hardcoded;
  - discovery driven by READMEs (a unit with no README silently disappears).
- **Conventions in stdblox-react's AGENTS.md** (possibly to adopt):
  - Prettier with `printWidth: 160`;
  - British spelling;
  - a file header comment;
  - JSDoc everywhere;
  - a `_` prefix for internal names;
  - `// #region` blocks;
  - a purpose comment at the top of every `describe`.
- **Other prior art:** `~/projects/ofzza/ngx-showcase` is an older Angular showcase library (not explored yet).

## 6. Pitfalls to address early (my notes)

- *(→ C10, D43/D48)* **Dependency resolution at build time.** Showcase pages import the user's source, so the user's `react`, `react-dom` and their versions must be used, not a copy bundled with the CLI. Two copies of React means broken hooks. The build must resolve framework packages from the showcased repo.
- *(→ C10)* **TS / path aliases in the showcased repo.** The bundler must respect the user's `tsconfig` paths, JSX settings, etc. Otherwise showcases that work in the repo fail in the showcase build.
- *(→ D5, C2 repo-level verification)* **Verification is per-directory only, but some rules are cross-cutting.** Example: "is this unit re-exported from the package entry?" needs repo-wide context. `verify` likely needs a context object (package info, entry files, the whole tree) and/or a whole-repo verification hook.
- *(resolved: D5)* **"Publicly exported" detection.** It needs a defined strategy: directory convention vs. static analysis of entry exports (TS compiler API). This affects both `check` and the hierarchy given to `build`.
- *(resolved: D40)* **Passing all file contents** for every directory can be expensive. Consider lazy content accessors.
- *(resolved: D13, D31)* **Interface extensibility and versioning.** The interface needs an `apiVersion`, optional capabilities, and input objects that can gain fields without breaking modules.
- *(resolved: D9, → C6)* **Loading custom modules and themes.** Custom modules are loaded with dynamic `import()` from a path. TS-authored modules need a loader (e.g. jiti/tsx) or must be prebuilt to JS.
- *(resolved: D46)* **Theme rule enforcement.** "No styling outside variables/classes" ideally needs tooling to enforce it (e.g. a stylelint rule banning literal values in module CSS), not just discipline.
- *(partly resolved: D8; caching → C11)* **Running tests during `build`.** It is slow and can fail. Decide whether a failing test blocks the build or is just displayed, and whether cached results can be reused.
