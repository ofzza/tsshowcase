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

  It outputs a report of the repository structure (all directories not ignored by `.gitignore`) with the issues found for each directory.
- **build**: builds a static, deploy-anywhere (e.g. GitHub Pages) showcase app from those resources. Output goes to a path argument, defaulting to `./showcase`. It uses the selected rendering module and theme.

**Modularity**

1. **Rendering modules** implement a clearly defined interface.
   - A module's default export is a class implementing the interface. The CLI imports it at runtime.
   - The CLI provides these inputs:
     - package info (`package.json` first; other repo types later);
     - the licence file contents;
     - the root `README.md`;
     - a component hierarchy, with for each unit: its README contents, its tests and their status, and the path to its showcase page;
     - room to extend later.
   - Modules also expose a **verification** method. It receives every non-ignored subdirectory together with its file list and file contents. It decides whether to ignore the directory and, if not, whether the directory meets the module's requirements.
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
| 1 | Requirements Q&A: until the requirements are clear | ✅ done (D1–D25); details continue in the chunk plans |
| 2 | Super-plan: partition the work into chunks, order them, mark which can be planned in parallel and which sequentially; iterate until approved | 🔄 **draft in `.plans/01-super-plan.md`, awaiting approval** |
| 3 | Chunk plans: one at a time, in super-plan order; iterate and approve each | ⏳ |
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
| D4 | **The CLI owns Vite**; modules contribute config | A module provides its framework plugins, app entry and runtime deps (e.g. a `getViteConfig(context)` method). An optional module `build()` override may be added later as an escape hatch. The showcased repo is never tied to Vite. Framework runtime packages (react, react-dom, …) **must resolve from the showcased repo**, while bundler plugins ship with the CLI/module |
| D5 | "Publicly exported" units are found by **static entry analysis** | The module analyses the package entry (`package.json` `exports`, then `index.ts`) with the TS compiler API and follows re-exports back to their source directories. This replaces stdblox's string match. **Consequence:** verification needs a repo-wide analysis step (e.g. `analyze(repo)`, which produces exported units / context) *before* per-directory `verify(dir, context)` |
| D6 | Configuration is an **optional config file plus CLI flags** | `tsshowcase.config.(ts\|js\|json)` holds the module, theme(s), module-specific options and output. CLI flags override the file |
| D7 | Themes: **several per build, with a runtime switcher** | The default follows `prefers-color-scheme`. The theme system defines a switcher class/slot that modules render |

| D8 | **Failing tests never block `build`** | Failures are shown per unit in the showcase. A `--fail-on-test-failure` flag gives strict mode (CI) |
| D9 | **How module/theme references resolve, in order:** (1) the name of a module/theme bundled with the CLI, (2) an installed npm package name, (3) a filesystem path | JS and TS are both allowed. TS is loaded at runtime with a loader (e.g. jiti), so custom module authors need no build step |
| D10 | **Adopt the stdblox code conventions** | Prettier `printWidth: 160`; British spelling; file header comments; JSDoc everywhere; `_` prefix for internals; `// #region` blocks; a purpose comment opening every `describe` |
| D11 | **Validation uses in-repo fixtures only** | Do not touch stdblox-react for now |
| D12 | **Licence file:** found case-insensitively as `LICEN[CS]E` with an optional `.md`/`.txt` extension, for each package in workspaces | Confirmed |
| D13 | **Contract versioning uses a semver range declared in code** | Every module, theme and test runner declares `static readonly apiVersion = '^1.2'`.<br>• **The interface has its own version**, independent of the CLI package version, plus an **interface changelog** ("added in 1.3: …"). It is exported as `IMPLEMENTED_API_VERSION`.<br>• The CLI checks `semver.satisfies(IMPLEMENTED_API_VERSION, apiVersion)` at load time and fails with a clear message.<br>• **Within a major version, changes are additive only.** Methods added after 1.0 are optional and feature-detected; modules ignore fields they don't know.<br>• The types package is versioned in step with the interface.<br>• **Rejected:** requiring a `package.json` with a peerDependency on the interface package for path-loaded modules |

| D14 | **This repo is an npm workspaces monorepo** | Separate packages: core CLI plus interface types, the React rendering module, themes, and the Vitest test runner. The CLI depends on the built-in packages so that lookup by name works (D9) |
| D15 | **Naming:** `@ofzza/tsshowcase` (the CLI, binary `tsshowcase`), with siblings `@ofzza/tsshowcase-<kind>-<name>` | See D27 for the full scheme |
| D16 | **v1 scope adds:** a `dev` action (Vite watch + HMR); an `init`/scaffold action (generate the stubs a module requires); **workspace (monorepo) showcased repos**; a machine-readable check report (`--format json`, non-zero exit on errors) | **Interface consequences:**<br>• the module needs a scaffold method;<br>• the input model must be multi-package from day one (a single-package repo = one package);<br>• `dev` needs a test strategy |
| D17 | **Node >= 20, ESM only** | |

| D18 | **In `dev`, tests run in watch mode** | Results are pushed live to the open showcase through the Vite dev server, so the test-runner interface must support watch and streamed updates |
| D19 | **Showcase hierarchy for workspaces:** repo README (home) → each package (its README and licence) → that package's units | The package level collapses for a single-package repo |
| D20 | **Layout variants are separate rendering modules** (e.g. a React per-page module vs a React single-page module) | They share a common React base (verification, embedding, helpers), probably as a shared package or a base class |
| D21 | **React verification:** base rules by default, opt-in rule presets | Base rules: own directory, README, test, showcase, exported from the entry. The stdblox headless pattern (`useX`/`renderX`/`X`) is an opt-in preset |

| D22 | **Only the React default module (`react-default`, D27) ships in v1** | The shared base (D26) makes further React modules cheap to add |
| D23 | **The CLI pre-renders markdown to HTML** | Output carries the documented theme classes, heading ids and rewritten links. Modules receive both the raw markdown and the HTML |
| D24 | **How repo-file links are rewritten:** the host (GitHub/GitLab/Bitbucket) is detected from `package.json` `repository` and the default branch from git; a configurable URL template overrides both | Links to unit READMEs become showcase routes |
| D25 | **Interactive playground is out of v1 scope** | The interface should leave room for it later |
| D26 | **The React base ships as a separate package from v1:** `@ofzza/tsshowcase-base-react` | **Base:** layout-agnostic Node-side parts (analysis, rules, scaffolding, Vite contribution, runner wiring) and runtime building blocks. It is not a loadable module.<br>**Default module:** layout only (shell, nav, routes, page composition). The user chose this over the recommended internal split |
| D27 | **Naming scheme `@ofzza/tsshowcase-<kind>-<name>`:**<br>• `-module-react-default` (built-in name `react-default`; renames "react-pages");<br>• `-theme-default` (`default-light`, `default-dark`);<br>• `-runner-vitest` (`vitest`);<br>• `-base-react`;<br>• `-api` | "single-page vs pages" was only an example of why several modules might exist, not a v1 requirement |
| D28 | **The theme catalogue lives in `-api` as importable typed constants** | Every class name and CSS variable name. Modules import the constants and never hardcode strings. C3 designs the hierarchy and implements the default themes |
| D29 | **Implementation order builds the "inputs" first** (`check`/`init` slice: C1→C2→C4→C6→C8→C9) | Lock down the inputs (scanning, analysis, data model) before moulding them into a themed showcase app |
| D30 | **The former C12 is split** into C12 (E2E acceptance), C13 (docs) and C14 (release) | Release scope is still being discussed |
| D31 | **Packages are versioned independently, with changesets** | The interface version (D13) stays separate from package versions |
| D32 | **Basic CI from C1** (lint, type-check, test, build on push/PR) | C14 extends it |
| D33 | **C14 scope:** CI extensions (Node matrix, E2E), pack-and-install smoke test, a **manual publish shell script** (dependency order, dry-run; no automated publish workflow for now), and a dogfood docs site on GitHub Pages | |

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

- **Dependency resolution at build time.** Showcase pages import the user's source, so the user's `react`, `react-dom` and their versions must be used, not a copy bundled with the CLI. Two copies of React means broken hooks. The build must resolve framework packages from the showcased repo.
- **TS / path aliases in the showcased repo.** The bundler must respect the user's `tsconfig` paths, JSX settings, etc. Otherwise showcases that work in the repo fail in the showcase build.
- **Verification is per-directory only, but some rules are cross-cutting.** Example: "is this unit re-exported from the package entry?" needs repo-wide context. `verify` likely needs a context object (package info, entry files, the whole tree) and/or a whole-repo verification hook.
- **"Publicly exported" detection.** It needs a defined strategy: directory convention vs. static analysis of entry exports (TS compiler API). This affects both `check` and the hierarchy given to `build`.
- **Passing all file contents** for every directory can be expensive. Consider lazy content accessors.
- **Interface extensibility and versioning.** The interface needs an `apiVersion`, optional capabilities, and input objects that can gain fields without breaking modules.
- **Loading custom modules and themes.** Custom modules are loaded with dynamic `import()` from a path. TS-authored modules need a loader (e.g. jiti/tsx) or must be prebuilt to JS.
- **Theme rule enforcement.** "No styling outside variables/classes" ideally needs tooling to enforce it (e.g. a stylelint rule banning literal values in module CSS), not just discipline.
- **Running tests during `build`.** It is slow and can fail. Decide whether a failing test blocks the build or is just displayed, and whether cached results can be reused.
