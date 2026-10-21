# Chunk plan C1: Monorepo foundation

> Status: **APPROVED** (2026-10-17), including amendment A1 (JSON schemas with tsschema sidecars, D62), re-approved the same day; see the end of this file. Amendment A2 (versioning from `0.1.0`, D80) re-approved 2026-10-18; see the end of this file. Editorial corrections: A1's wording updated after D66 moved the config schemas to the CLI package (C2 review F42); the glossary seed term "region class" replaced by "shell (shadow root)" after D89 (C3 planning). No change in scope. Amendment A3 (from C4 planning: the private `-test` package, fixture ignore rules; D104, D109, D107) re-approved 2026-10-18 with the C4 plan; see the end of this file. Note: C3 later adds an eighth package, `-lint` (D88), and C4 a private ninth, `-test` (D104); "7 packages" below refers to C1's own scope (01b K29). Decisions referenced as `D<n>` are in AGENTS.md §3. **Amendment A4 (the representation redesign, `08b-representation-redesign.md`, D160–D200) **approved 2026-10-20** (review `01e-c1-a4-review.md`):** §5 and "Recorded for later chunks": no `ignore: ['/fixtures/']`; §8 the dogfood layout under the representation rules; §9 the layout/glossary wording; §10 trigger rows, single source per fact, the automated checks; §1/§2/§4 `-lint`, the skeleton comment, the entry-file ESLint rule, type tests, extra and opt-in Vitest projects; see "Amendment A4" at the end. **Amendment A5 (the TS template as the source, D203) approved 2026-10-21** (`01g-c1-a5-review.md`): the implementation imports from the template by hand, interactively with the user, and logs the import; see "Amendment A5" at the end. **Wording from plan 6 (D205, `05b-c5-redesign-review.md` Fix 27):** §5 gotcha 2 re-annotated for `vcs.ref: CURRENT`. **Wording from plan 7 (C6 redesign amendment, `06b-c6-redesign-review.md`; approved 2026-10-21):** §5 the CLI package lists three schema copies.

## Goal

An empty but fully wired npm-workspaces monorepo. Every later chunk drops code into a package that already builds, lints, type-checks, tests and runs in CI.

C1 adds **no product code**: no contracts, CLI commands, themes or analysis. Those belong to C2 onwards.

**Baseline (D52, D203):** the user's TypeScript project template at `~/projects/ofzza/.templates/ts` (the repo `https://github.com/ofzza/.templates`) is **the source** for tooling, conventions and docs: the user's projects share one lineage. Everything below mirrors it, adapted from a single package to a monorepo. Deviations are called out explicitly; differences this plan doesn't call out yet (listed in `01f-c1-a5-template-notes.md`) are settled with the user during the import and recorded in `TEMPLATE.md`.

**Importing from the template (A5):**

- **First step of the implementation:** copy from the template by hand what suits this repo, file by file. What needs merging or consolidating with what already exists (`AGENTS.md`, `README.md`, `CLAUDE.md`, this plan's adaptations) is done **interactively with the user**. `REQUIREMENT.md` and `.plans/` are never touched. Starting checklist: `.plans/01f-c1-a5-template-notes.md` (the template compared with this plan, incl. differences this plan doesn't call out yet), re-checked against the template's current commit.
- **The import log, written right after the import:** the template commit used, and per template file whether it was imported as is, merged or changed (with what and why), or left out (and why). It is what later lets anyone diff the template from that commit and consider bringing changes in. It lives in **`TEMPLATE.md`** at the root (TP2), written for humans: which template repository (URL) and template directory, which commit (and date) was used, and what was imported and how, file by file; `AGENTS.md` ("Project") links to it so agents find it, and a trigger row (§10a) covers template changes.

## Deliverables

### 1. Repository layout

```
.changeset/                    changesets config + README (D31)
.github/
  workflows/ci.yml             CI (D32), Node matrix [22, 24] (D54)
  dependabot.yml               weekly npm + github-actions updates against develop
.vscode/
  extensions.json              recommended: ESLint, Prettier, TODO Highlight, Twoslash Query Comments (as the template)
  launch.json                  Vitest debug configs: all tests, unit tests, current file (as the template)
  settings.json                format on save with Prettier, workspace TypeScript
fixtures/                      showcased-repo fixtures, isolated from the workspace (see §5)
  README.md
packages/
  tsshowcase-api/                         @ofzza/tsshowcase-api
  tsshowcase/                             @ofzza/tsshowcase                  (the CLI; bin: tsshowcase)
  tsshowcase-theme-default/               @ofzza/tsshowcase-theme-default
  tsshowcase-testrunner-vitest/           @ofzza/tsshowcase-testrunner-vitest
  tsshowcase-base-vite/                   @ofzza/tsshowcase-base-vite
  tsshowcase-base-react/                  @ofzza/tsshowcase-base-react
  tsshowcase-module-react-default/        @ofzza/tsshowcase-module-react-default
  tsshowcase-lint/                        @ofzza/tsshowcase-lint             (added by C3, D88: lint rules and presets)
  tsshowcase-test/                        @ofzza/tsshowcase-test             (private, added by C4: test helpers, A3)
scripts/
  fixtures.mjs                 install / clean the fixtures' own dependencies
  readme-samples.mjs           type-checks the ts/tsx code samples in every README (§10)
.editorconfig  .gitignore  .nvmrc  .prettierrc  .prettierignore
eslint.config.js  tsconfig.base.json  tsconfig.build.json  tsconfig.json  vitest.config.ts
package.json  package-lock.json  LICENSE  README.md  CONTRIBUTE.md  TEMPLATE.md (A5: the template import log)
AGENTS.md  CLAUDE.md  REQUIREMENT.md  .plans/
```

Each package directory is named after the full npm package name without the `@ofzza/` scope (D57): `packages/tsshowcase-<kind>-<name>/`, and `packages/tsshowcase/` for the CLI. The npm names follow D27. `<pkg>` below means this directory name.

**`.gitignore`** mirrors the template's file, adding only what the monorepo needs (`fixtures/*/node_modules/` is already covered by `node_modules/`; `showcase/` build output is ignored once C11 exists). It keeps:

- the `.vscode/*` ignore with the four re-included files (`settings.json`, `tasks.json`, `launch.json`, `extensions.json`);
- the scratch-file convention `*.gitignore.*` / `*.gitignore/`;
- the `*.debug.test.ts` scratch-test convention (§4);
- `*.tsbuildinfo`.

### 2. Package skeletons

Every package gets the same files:

```
packages/<pkg>/
  package.json
  tsconfig.json          type-check config: all of src/ including tests, noEmit
  tsconfig.build.json    emit config: composite, src → dist (`include` also has src/**/*.json, A1), excludes tests/showcases, references its workspace deps
  README.md              template structure: title, purpose, Get / Usage / Development / Contributing (Usage "under construction")
  LICENSE                copy of the root LICENSE (npm only packs the package's own licence file)
  src/
    index.ts             entry: file header + `export {};` (no exported symbols yet)
    index.test.ts        wiring test: the entry imports cleanly
```

**`package.json` fields:**

- `name`, `version: 0.1.0` (A2, D80), `description`, `keywords`, `license`, `author`;
- `repository` with `directory: packages/<pkg>`, plus `homepage` and `bugs`;
- `type: module`, `engines.node: ">=22.12"` (D49; a deliberate _runtime_ floor, see §11), `sideEffects: false` (except where a later chunk needs CSS side effects);
- `exports`: `{ ".": { "source": "./src/index.ts", "types": "./dist/index.d.ts", "default": "./dist/index.js" }, "./package.json": "./package.json" }`; `main` and `types` mirror it for older tooling (as the template);
- `files: ["dist"]`, `publishConfig.access: public` (the CLI package also lists its three config schema copies at the package root (the third, `directory-config.schema.json`, from plan 7), made by its own `build` script: C6 §5, wording-only note from C6 planning);
- scripts: `clean`, `build` (with `prebuild: clean`, as the template).

**The CLI package also gets** `bin: { "tsshowcase": "./dist/bin.js" }`, with a `src/bin.ts` stub that prints the version and "not implemented yet". It only prints when executed directly, never on import (the template's `import.meta.url` vs `process.argv[1]` guard), which keeps `sideEffects: false` truthful. C9 replaces it.

**Internal dependencies are declared now**, following the super-plan package table. Ranges are `^0.1.0` (A2), and changesets bumps them:

| Package                 | Depends on                                                                                                |
| ----------------------- | --------------------------------------------------------------------------------------------------------- |
| `-api`                  | –                                                                                                         |
| `-theme-default`        | `-api`                                                                                                    |
| `-testrunner-vitest`    | `-api`                                                                                                    |
| `-base-vite`            | `-api`                                                                                                    |
| `-base-react`           | `-api`, `-base-vite`                                                                                      |
| `-module-react-default` | `-api`, `-base-react`                                                                                     |
| CLI                     | `-api`, `-theme-default`, `-testrunner-vitest`, `-module-react-default` (built-ins reachable by name, D9) |
| `-test` (private, A3)   | –; a `devDependency` of the packages whose tests use it, never a project reference (references cover runtime dependencies only) |

Third-party runtime deps (Vite, React plugin, unified, …) are **not** added in C1. Each chunk adds its own. Build, lint and test tooling lives in the root `devDependencies` only.

### 3. TypeScript

- `tsconfig.base.json` holds the shared options:
  - `module`/`moduleResolution: nodenext`, `target: es2023` (safe on Node 22), `lib` to match;
  - `strict`, `verbatimModuleSyntax`, `isolatedModules`, `skipLibCheck`, `forceConsistentCasingInFileNames`, `resolveJsonModule`, `allowArbitraryExtensions` (A1: tsschema sidecars type JSON imports);
  - `declaration`, `declarationMap`, `sourceMap`;
  - `customConditions: ["source"]`;
  - `types: ["node"]`.
  - Browser-side packages add the `dom` lib and `jsx: react-jsx` in their own tsconfig when a later chunk needs them.
  - **Deviation from the template:** `nodenext` instead of `bundler`. Our packages run in Node unbundled, and `nodenext` makes `tsc` _enforce_ the `.js` extensions the template's gotcha asks for.
- **Cross-package imports resolve to source (the `source` export condition).** Type-checking, tests and the editor therefore work without building first.
  - `tsc -b` still builds in dependency order through project references.
  - This also dogfoods the "map built entries back to source" case that C8's analysis must handle.
- **Build:** the root `tsconfig.build.json` is a solution file that references every package's `tsconfig.build.json`. `npm run build` runs `clean` first (the template's guard against stale output from renamed or removed files), then `tsc -b tsconfig.build.json`.
- **Type-check:** package code and tests are type-checked by Vitest's typecheck pass (§4). The root `tsconfig.json` covers root-level files (configs, scripts), checked by `ci:typecheck`.
- **No `paths` aliases** (template gotcha: `tsc` doesn't rewrite them on emit).
- **TypeScript version:** pinned to 6.0.x (D50).
- **`@types/node` pinned to `^22`**, the floor major (D49). Dependabot ignores its major updates, so it is bumped by hand together with the floor.

### 4. Lint, format, test

- **Prettier:**
  - the template's `.prettierrc` (printWidth 160, single quotes, 2 spaces, `arrowParens: always`);
  - **checks the whole repo, Markdown included (D55)** (deviation: the template checks `src` only);
  - `.prettierignore` covers `dist`, `node_modules`, lockfiles, `.changeset/*.md` (generated), `**/*.d.json.ts` (generated tsschema sidecars, A1) and `REQUIREMENT.md` (verbatim, never edited).
- **ESLint (flat config, the template's file):**
  - `@eslint/js` recommended (a direct devDependency, as the template's gotcha requires), `typescript-eslint` recommended (type-aware for `packages/*/src`), and `eslint-config-prettier`;
  - the template's rule tweaks (`_`-prefixed unused vars, args and catch bindings allowed; `no-explicit-any`, `ban-ts-comment`, `no-empty-object-type` off);
  - `.gitignore` included as ignores; `fixtures/**` ignored.
  - **entry files re-export by name only (A4):** a `no-restricted-syntax` rule bans `ExportAllDeclaration` (`export * from …`) in every package's entry files (the sources of its `exports` subpaths), so the public API is listed explicitly (§8).
  - `eslint-plugin-react-hooks` is scoped to `base-react` and `module-react-default` and added in C10, when there is React code to lint.
  - Stylelint and the D46 styling rules arrive in C3.
- **Vitest (the template's setup, multiplied by package):**
  - one root `vitest.config.ts` with `reporters: ['tree']` and `test.projects`.
  - **`unit:<pkg>`**, one per package, named after its directory (e.g. `unit:tsshowcase-api`):
    - runs `src/**/*.{test,unit.test,spec,unit.spec}.{ts,tsx}`;
    - **type-checks the same files plus the type-only test files** through `tsc` with the package's `tsconfig.json` (`typecheck.enabled`; `typecheck.include` adds `src/**/*.test-d.{ts,tsx}`, A4, which the runtime pattern doesn't match). Type assertions (`expectTypeOf`, `@ts-expect-error`) and type-only test files (`*.test-d.ts`) are ordinary test files and fail as tests;
    - `typecheck.spawnTimeout: 10000` (the template's Vitest 5 race workaround);
    - like the template, every test file is reported twice, once per pool (gotcha documented);
    - Node environment by default (C10 switches the React packages to jsdom).
  - **Additional projects per package (A4)** where a part of it needs a different environment or tsconfig, named `unit:<pkg>:<part>` (e.g. `unit:tsshowcase-base-vite:runtime` for a `dom`-lib runtime with jsdom tests), each with its own `include` and `typecheck.tsconfig`; the chunk that needs one specifies it (C15, C10).
  - **`debug`**, one project across all packages: runtime only, no type checking. It also matches `*.debug.test.ts` scratch tests, which are gitignored. **Opt-in (A4):** `vitest.config.ts` includes it only when `TSS_VITEST_DEBUG=1` is set, so a plain `vitest`, CI and tsshowcase's own test listing and runs (D172) see only the `unit:*` projects and never list or run each test twice.
  - Workspace imports resolve via the `source` condition, so tests never need a build.
- **`.vscode/launch.json`:** the template's three configs ("Debug ALL Test Files", "Debug UNIT Test Files" → `--project "unit:*"`, "Debug Current Test File" → `--project debug`); the configs using the `debug` project set `TSS_VITEST_DEBUG=1` (A4).
- **Root scripts** (with `npm-run-all2`, as the template):
  - `clean`, `build` (`prebuild: clean`), `dev` (`tsc -b --watch`), `lint` (ESLint), `format` / `format:check` (Prettier), `typecheck` (root files), `test` (`npm-run-all test:*`; `test:unit` → `vitest run --project "unit:*"`);
  - `debug` → `TSS_VITEST_DEBUG=1 vitest --project debug` (A4; deliberately not a `test:*` script, so `npm test` and CI never run it);
  - `ci` = `npm-run-all ci:*` → `ci:build`, `ci:typecheck`, `ci:eslint`, `ci:prettier`, `ci:test-unit`, `ci:readme-samples`, `ci:links` (§10), `ci:schemas` (A1);
  - `schemas` → `tsschema generate --mode sidecar "packages/*/src/**/*.schema.json"` (A1); `ci:schemas` → the same with `--check`. `build` runs `schemas`, then `tsc -b`, then `scripts/copy-sidecars.mjs` (A1);
  - `prepare: npm run build` (root only, as the template), `prepublishOnly` guard noted for C14;
  - `fixtures:install`, `fixtures:clean`;
  - `changeset` (the changesets CLI).
  - The template's script wiring rule applies: a new `test:<name>` also needs a matching `ci:test-<name>`.

### 5. Fixtures convention

- `fixtures/<name>/` is a self-contained showcased repo with its own `package.json`. Where it needs real deps (`react`, `vitest`), it also has its own committed `package-lock.json`.
- **Isolated from the workspace:**
  - not in `workspaces` (that only lists `packages/*`), so no hoisting into or out of it;
  - excluded from ESLint, the TS configs and the Vitest projects. Prettier skips only their `node_modules` and the intentionally broken `fixtures/broken-*/` fixtures (A3), so fixture sources are still formatted.
- `scripts/fixtures.mjs install` runs `npm ci` in every fixture that has a lockfile. `clean` removes their `node_modules`. CI runs `fixtures:install` before tests.
- **Known gotcha: fixtures live inside this repo.** Run in place, a fixture sees this repo's state, which breaks three kinds of test:
  1. **A missing dependency still gets found.** Node resolves packages by walking _up_ the directory tree, so a fixture without `react`, `typescript` or `vitest` still finds the copy in this repo's root `node_modules`. A fixture's _own_ copies are fine, because the nearest `node_modules` wins. Affects:
     - "React must resolve from the showcased repo, with a clear error when it has none" (C10);
     - "use the showcased repo's own TypeScript" (C8);
     - "no Vitest installed → tests unavailable" (C7).
  2. **Git state leaks in.** (A3: since C4 reads no `.gitignore` above the scan root and runs no git (D103, D110), this mostly doesn't affect C4. *Since D205 (plan 6) it does again for `vcs.ref: CURRENT`, the default: a fixture scanned in place finds this repo's `.git` above it and pins links to this repo's commit; a copied fixture has no `.git` and fails once VCS links are produced. Tests needing VCS links set `vcs.ref` explicitly or write a fake `.git` through `copyFixture`'s `files` (C5, C11, C12; `05b` Fix 27).*) Scanning and git info (C4) would find this repo's `.git` and root `.gitignore`. That makes "not a git repo", a GitLab/Bitbucket/missing remote, a detached HEAD, a shallow clone and a missing `origin/HEAD` untestable. Nested `.git` directories can't be committed either.
  3. **Tests that write files.** `init` writes stub files (C8/C9). `build` writes an output dir (C11), and its output guard needs a non-empty directory tsshowcase didn't create. These tests must not change the committed fixtures, or collide with each other when run in parallel.
- **The fix: a test-only `copyFixture(name, options) → { dir, cleanup }` helper, specified and implemented in C4** (the first chunk whose tests need it):
  - copies `fixtures/<name>/` into a fresh temp directory under `os.tmpdir()`. Nothing above it has a `node_modules` or `.git`, so a missing dependency really is missing;
  - skips `node_modules` when copying. Optionally links the fixture's already-installed `node_modules` into the copy, so nothing is reinstalled per test;
  - optional `git` settings: `git init` plus a commit, then set up the state the test needs (remote URL or none, branch name, detached HEAD, no `origin/HEAD`, …);
  - `cleanup()` removes the temp directory (called from `afterEach`).
  - Tests that only read a fixture and are unaffected by the three cases above may use `fixtures/<name>/` in place.
  - It lives in the private `-test` package (C4, D104; A3).
- C1 ships only `fixtures/README.md` describing this. The first fixtures arrive with C4.
- When `tsshowcase check` dogfoods this repo (C9), `fixtures/` needs **no** configuration (A4): it lies outside every package (the workspace root is never a package and `workspaces` lists only `packages/*`), so it is never in scope (D165). (Before A4: config `ignore: ['/fixtures/']`.)
- **Fixture ignore rules (A3, D109):** fixtures carry their own ignore rules in `.tsshowcaseignore` files, which this repo's git doesn't apply, so everything in a fixture gets committed. Fixture content this repo's own `.gitignore` would catch (e.g. a fixture `dist/`) is re-included with `!` rules, at the root, in `fixtures/.gitignore` or in the fixture itself.

### 6. Versioning: changesets (D31)

- `@changesets/cli` with `.changeset/config.json`:
  - `access: public`, `baseBranch: develop`;
  - independent versioning (no `fixed` or `linked` groups);
  - `updateInternalDependencies: patch`;
  - `privatePackages: { version: false, tag: false }`, so the private `-test` package is never versioned or tagged (A3).
- No publish automation (D33: a manual publish script arrives in C14).
- **Open for C14:** the `source` export condition points at `src/`, which isn't packed. The C14 pack step either strips the condition or also packs `src/`. The C14 smoke test catches either mistake.

### 7. CI (D32)

`.github/workflows/ci.yml`, the template's workflow:

- triggers: push and pull requests on `master` and `develop`;
- cancels superseded PR runs only, never `master`;
- `permissions: contents: read`;
- **Node matrix `[22, 24]` with `fail-fast: false` from C1 (D54)**. C14 adds Node 26;
- steps: checkout → setup-node (npm cache, also keyed on the fixtures' lockfiles) → plain `npm ci` (never `--ignore-scripts`, template gotcha) → `npm run fixtures:install` → `npm run ci`.

`.github/dependabot.yml`, the template's file:

- weekly npm and github-actions updates targeting `develop`;
- devDependencies grouped into one PR;
- `@types/node` majors ignored;
- fixture lockfiles get their own entries once fixtures exist (C4+), ignoring majors.

Branch protection (required checks `CI / Node 22`, `CI / Node 24`) is a GitHub setting, documented in AGENTS.md for the user to apply.

### 8. Dogfooding (D41)

- C1 packages export **no symbols** (`export {};`), so they satisfy the convention trivially. The convention is documented now and binds every later chunk, every package included (also `-lint`, added by C3).
- **This repo's layout under the representation rules (A4; D161–D163, D171, D173, what `tsshowcase check` enforces):**
  - **One exporting file per concept:** each public concept is declared in its own file, `packages/<pkg>/src/<area>/<Concept>.ts(x)`, named after its main export in that export's case (`ShowcaseModule.ts`, `issue.ts`). Its symbols are what `check` requires resources for, per the kind requirements (D162, C2 §5.1). A symbol's kind follows what the exported value is, not how it is declared (D200): an arrow function in a `const` is a function and needs tests.
  - **One `README.md` in every directory that directly contains exporting files,** documenting each of them. A directory README covers only its own directory's files (D173), so a nested directory with exporting files needs its own; directories without exporting files (e.g. `-api`'s `src/compat/`) need none. Extra per-concept docs, when a README gets long, are `<Concept>.md` next to the file (primary, by name).
  - **Tests and showcases named after the file they cover:** `<Concept>.test.ts(x)`, `<Concept>.showcase.tsx` (by name, primary). With exactly one exporting file in a directory, other resources there belong to it too (strong), but naming them after the file is the convention. Names match exactly (D130); avoid exporting files whose names differ only in extension (`config/same-base`) or only in case.
  - **Areas** are ordinary directories (`src/version/`, `src/module/`). A `.tsshowcase.jsonc` that is `true`/`{}` or sets a node key (`paths`, `name`, `slug`, `order`) makes its directory a menu level (an indirect node, whose page shows the directory README, D173, D188); one with only `collections`, `kinds` or `exempt` creates no node. In `src/` (the single literal scope path) it configures the package root instead (D179). Otherwise the tree is sparse.
  - **Concepts split over several files** (e.g. a component, its hook and its render function) are grouped with a `collections` entry in that directory's `.tsshowcase.jsonc` (D190; this alone creates no menu level). The collection is named after the concept (`Button`), and its docs, tests and showcases after that key (`Button.md`, `Button.test.tsx`, `Button.showcase.tsx`); members' own name-matched resources (e.g. `useButton.test.ts`) count for the collection too.
  - **Entry files only re-export, by name.** Every `exports` subpath's source file (`src/index.ts`, `src/runtime/index.ts`, …) lists the package's public symbols explicitly (`export { ShowcaseModule, type ModuleInit } from './module/ShowcaseModule.js'`), so what is exported is visible and reviewed in one place; `export *` is not used in entry files (an ESLint `no-restricted-syntax` rule enforces it, §4), and default exports are re-exported explicitly. Internals may then live in any file; a `_` prefix is naming only, it never hides an export. **No exporting files directly in `src/`:** public concepts live in area directories (a lone exporting file there would also claim the package's wiring test `src/index.test.ts` through the single-file rule). **Every `exports` subpath carries a `source` condition** (§2), so the analysis maps it to source. `src/bin.ts` exports nothing. No per-concept barrels (`<Unit>/index.ts`) and no `_`-prefixed internal directories are needed: files that declare no public symbol need nothing, wherever they are.
  - **Public API from dependencies is declared locally:** a type or value from another package that is part of our public API gets a local declaration (`export type JsonSchema = TSSchemaJSONSchema;`), never a bare re-export, so it attaches here and is checked and documented here (declarations in `node_modules` or another workspace package are never attached, D171).
  - **JSON files:** a JSON file is an exporting file only when its default export is **re-exported from an entry** (e.g. the CLI's `configSchema`, C6); it then needs docs like any constant (D171, D162). JSON files that are only imported (`package.json`, internal options schemas) are not; JSON `exports` subpaths are not analysed (08a U9, carried to C8); tsschema sidecars (`*.d.json.ts`) are never exporting files.
  - **Directory configs** in this repo are `.tsshowcase.jsonc` files only, placed in the directory they configure: JSONC needs no extra tooling (editors and Prettier infer the parser, no code is executed) and allows comments, e.g. the reason for a `kinds` or `exempt` entry. No `directories` entries in this repo's configs (one place per fact; overlaps would raise `config/shadowed`). The other forms and the configuration they need are documented in C13.
  - **This repo's root `tsshowcase.config.json`** (written in C9): `scope: ['src']`, so only `src/` is checked, verified and reported and package-root files (manifests, tsconfigs, build output, `-theme-default`'s `css/`) stay out of scope (the package `README.md` and `LICENSE` are home docs either way, D185); `strict: ['check', 'warnings']`, so warnings fail the dogfood check locally and in CI alike (D164, D175). The private `-test` package keeps `{ "skip": true }` (A3, D104). `fixtures/` needs no configuration: it lies outside every package (D165).
  - **Root documents** (`README.md`, `LICENSE`, `CONTRIBUTE.md`, `AGENTS.md`, `CLAUDE.md`, `REQUIREMENT.md`, later `PUBLISH.md`, and each package's `README.md`, `LICENSE`, `CHANGELOG.md`) are home docs (D185); `check` validates their links and anchors like any other doc's (D183), which `ci:links` keeps clean. Should the verbatim `REQUIREMENT.md` ever fail it, a narrow `rules` override, or an `ignore` of that one file, is the remedy.
  - **Build exclusions:** every test form (`*.test.*`, `*.unit.test.*`, `*.spec.*`, `*.test-d.*`, `*.debug.test.*`), `*.showcase.tsx` and `src/compat/` are excluded from `tsconfig.build.json`, so they never reach `dist`; directory configs are dotfiles, which TypeScript's wildcard `include`s skip, and `files: ["dist"]` keeps them out of the package.

### 9. Documentation (D53)

The template's three-way split, adapted:

- **`AGENTS.md`** (agents). §1–§6 keep their role as the process and decision log. It gains the template's agent sections:
  - **Keeping documentation and tests current:** the trigger table, extended for this repo (§10);
  - **Project:** what tsshowcase is, licence, ESM only, `files` allowlist, "toolchain last verified against" line, and a link to `TEMPLATE.md` (A5: the template this repo is based on and the import log);
  - **Branching** and **Commits** (the template's text: commit to the current branch, never to `master` or a detached HEAD, propose feature branches, `develop` trunk / `master` stable, no AI attribution);
  - **Layout:** the tree above, what each package is for, and the representation convention (§8, A4);
  - **Commands:** every root script, plus the script wiring rule;
  - **Testing:** the Vitest projects, the typecheck double-reporting, `spawnTimeout`, the `debug` project and scratch tests, the fixtures convention (§5, including when to use `copyFixture()`);
  - **Code style (D10 + the template):**
    - Prettier as configured, whole repo including Markdown; British spelling;
    - every file opens with a descriptive header comment;
    - JSDoc on every exported symbol, with a third-person summary ("Gets …", "Checks if …", "Represents …"), using only `@param` and `@returns` (never `@example`, `@template` or `@see`);
    - internal symbols are `_`-prefixed and still get JSDoc;
    - `// #region Name` / `// #endregion` sections;
    - generic parameters `T`-prefixed and descriptive (`TArray`, `TObject`), including `infer` bindings, never shadowing an outer one;
    - in conditional-type chains, a `//` comment line before each branch;
    - one top-level `describe` per test file, named after its module; `it` names are capitalised verb phrases;
    - every `describe` callback opens with a purpose comment (what is tested, and how). A nested `describe` says only what its parent's doesn't;
    - an `it` gets a comment only when its title and body don't explain why it exists (regression, edge case, gotcha guard linking `See AGENTS.md › <gotcha>`, non-obvious assertion);
    - tests beside the code as `*.test.ts(x)`; relative imports with explicit `.js` extensions;
    - rules deliberately off: `no-explicit-any`, `ban-ts-comment`, `no-empty-object-type`;
  - **Gotchas and known issues** (§11);
  - **Continuous integration:** the workflow, Dependabot, and the branch-protection note;
  - **Packages:** one line per package linking to its README (which holds the export list; see §10).
- **`CONTRIBUTE.md`** (contributors): reporting issues and the contribution workflow (linking the ofzza onboarding guides, as the template), branching, the project structure, and a **glossary**. The glossary is seeded now with the terms already in use (A4: the representation terms of 08b §1.1 / C2 §13 replace "unit"): symbol, exporting file, resource, association, node, directory config, collection, in scope, showcase, showcasing component, showcased repo, rendering module, base package, theme, theme catalogue, shell (shadow root), test runner, fixture, interface version.
- **`README.md`** (users): a short stub now (what tsshowcase is, "under construction", Development and Contributing sections). C13 writes the real one.
- **Package READMEs:** the template's structure (§2).
- **`PUBLISH.md`:** arrives with C14, mirroring the template's checklist (see "Recorded for later chunks").
- **`LICENSE`:** MIT, ofzza (D51).

### 10. Keeping docs, tests and showcases current at scale (D56)

The template relies on a trigger table and discipline. This repo will have dozens of area READMEs, tests and showcases across its packages, so the plan is to **keep the trigger table** and **back it with automation wherever "out of date" can be detected mechanically**.

**a) The trigger table, extended.** The template's rows are kept, and these are added:

| A change that …                                         | … requires updating                                                                                                                                          |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| adds an exporting file (a public concept)               | its directory's README (or a `<Concept>.md`), a test named after it and a showcase where its kinds require them (D162); the named re-export line in the package's entry file; the package README's export list (A4) |
| changes a symbol's signature, props, options or semantics | the area README (or `<Concept>.md`), tests (and their purpose comments) and showcase; every README sample using it                                         |
| removes, renames or moves an exporting file or a symbol | the entry's named re-export line, the package README's export list, the README of the old and of the new directory, every README linking to it, its test and showcase file names, and any directory config listing it (`paths`, `collections`, `kinds`, `exempt`) |
| adds an area directory                                  | its `README.md`; a `.tsshowcase` if it should be a menu level (A4)                                                                                           |
| changes a directory config or a collection              | the docs of the nodes it creates or changes; `tsshowcase check` passes (A4)                                                                                   |
| adds or changes a `kinds` or `exempt` entry             | a JSONC comment next to it stating the reason (they weaken the check; the `check` summary counts exemptions, D191) (A4)                                      |
| adds a showcasing component (a new export in a `*.showcase.tsx`) | a meaningful name or title for it (its `meta`, D195); the smoke render covers it automatically (A4)                                                     |
| adds or changes a module's `static kinds`               | the kinds' documentation in that module's README and the C13 docs of what each kind requires (D162) (A4)                                                     |
| changes an `-api` contract                              | the interface changelog (D13: additive within a major), the authoring docs, and every in-repo implementer (built-in module, theme, runner)                   |
| changes a theme catalogue class or variable (from C3)   | the catalogue docs, the default themes, and the module/runtime code using it                                                                                 |
| changes a CLI flag, config key or output format         | the CLI reference in the root README and the config schema docs                                                                                              |
| adds a package                                          | the layout and packages list in AGENTS.md, the root `tsconfig.build.json`, Vitest projects, the dependency table, CONTRIBUTE.md's structure; its own `tsshowcase.config.json` where needed (`skip`, a `scope` override; A4) |
| changes a fixture                                       | the tests using it and `fixtures/README.md`                                                                                                                  |
| the TS template (`~/projects/ofzza/.templates/ts`) changes, or more of it is brought in | diff the template from the commit in `TEMPLATE.md`, decide each change with the user, update `TEMPLATE.md` (commit and per-file entries) (A5, D203) |
| changes a package's public behaviour at all             | a changeset (`npm run changeset`)                                                                                                                            |

**b) Single source per fact.** Each fact is written once and linked from elsewhere, so there are fewer places to update:

- a package's export list lives in that package's README only (AGENTS.md links to it, and doesn't repeat it as the template does);
- a concept's documentation lives in its area README (or its own `<Concept>.md`) only (D163, A4). Package READMEs link to it rather than repeating it.

**c) Automated checks, added to `npm run ci`:**

| Check                                                                                                                                                                                                                                                                                                 | Catches                                                                     | Chunk  |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------ |
| `ci:readme-samples`: `scripts/readme-samples.mjs` extracts every ` ```ts `/` ```tsx ` block from every `README.md`, and type-checks them against the workspace (imports of `@ofzza/tsshowcase-*` resolve to source via the `source` condition). A block can opt out with a `nocheck` fence attribute. | README samples drifting from the API (automates the template's manual rule) | **C1** |
| `ci:links`: `remark-validate-links` over every Markdown file (offline; relative files and heading anchors).                                                                                                                                                                                           | Links broken by a rename, move or removed heading                           | **C1** |
| Representation (every public symbol represented with the docs, tests and showcases its kind requires; framework rules): `tsshowcase check` on this repo (D41, D174, A4)                                                                                                                                                                                                  | A public symbol missing its docs, tests or showcase                         | C9     |
| Showcase smoke render: every showcasing component of every `*.showcase.tsx` renders without throwing (stdblox's `showcases.test.tsx` pattern; per (file, export), D195, A4)                                                                                                                                                                                       | Showcases broken by an API change                                           | C10    |
| PR changeset check: `changeset status --since=origin/develop` on pull requests that touch `packages/`; **shipped disabled until the first publish** (A2, D80)                                                                                                                                                                                                 | Forgotten changesets                                                        | **C1** |

What stays manual (and is said so in the trigger table's intro): whether README _prose_ and showcase _content_ still describe the behaviour accurately. Reviews and the purpose-comment rule cover these.

### 11. Gotchas carried over from the template (documented in AGENTS.md)

- Never `npm ci --ignore-scripts` (native bindings for `unrs-resolver` / `@parcel/watcher`).
- `prepare` builds during install, so `npm ci` compiles once and `ci:build` compiles again. Accepted.
- `build` cleans first, so renamed or removed files never leave stale output to publish.
- Relative imports need explicit `.js` extensions (here `nodenext` enforces it).
- `files: ["dist"]` is an allowlist; check with `npm pack --dry-run`.
- The `exports` map seals deep imports. Add a subpath only for something that must be importable on its own.
- No `paths` aliases.
- Anything in `dependencies` installs for every consumer. Tooling stays in root `devDependencies`.
- `@eslint/js` must be a direct devDependency.
- `@types/node` is pinned to the floor major and bumped by hand together with it.
- The `engines` field states the **runtime** floor (D49): Vitest 5 and Vite 8 run inside tsshowcase on users' machines. This is deliberately different from the template's advice not to encode the _dev toolchain's_ floor in `engines`.
- Every test file is reported twice (runtime + typecheck), and a test file with no runtime suite needs `--passWithNoTests`.
- `npm-run-all2`, not `npm-run-all`.
- Plus this repo's own: the fixture walk-up (§5), the `source` condition at pack time (§6), and **sidecars not reaching `dist`** (A1).

## Acceptance criteria

1. `npm ci && npm run ci` passes on a clean checkout on Node 22 and 24: build, root type-check, ESLint, Prettier (whole repo), the 7 wiring tests (runtime + typecheck), README samples, links.
2. `npm run build` emits `dist/` for all 7 packages in dependency order. `node packages/tsshowcase/dist/bin.js` prints the stub message.
3. A test in one package that imports another workspace package runs **without** a prior build (via the `source` condition). Checked once by hand during implementation.
4. A deliberately broken type assertion in a test fails `npm test` (the template's sanity check), and so does one in a `*.test-d.ts` file (A4). With `TSS_VITEST_DEBUG` unset, `vitest list` shows only `unit:*` projects (A4). Reverted afterwards.
5. A deliberately broken README sample fails `ci:readme-samples`, and a broken relative link fails `ci:links`. Reverted afterwards.
6. `npm run fixtures:install` succeeds with zero fixtures. A temporary throwaway fixture confirms it is ignored by ESLint, TS and Vitest, and that Prettier skips only its `node_modules`.
7. `npx changeset status` works.
8. The three `.vscode/launch.json` configs start a debug session.
9. The CI workflow passes on GitHub after the push.
10. AGENTS.md, CONTRIBUTE.md, the root README and the 7 package READMEs exist with the content listed in §9.

## Out of scope

Contracts (C2), theme design and stylelint (C3), all runtime dependencies, React/jsdom test setup (C10), Node 26 in CI, E2E, pack smoke test, PUBLISH.md and publishing (C14), and the self-check in CI (C9).

## Recorded for later chunks

- **C14:**
  - the consumer smoke test type-checks each packed package under **both** `nodenext` and `bundler` resolution, and imports it at runtime (as the template's PUBLISH.md does);
  - a `prepublishOnly` guard so a failing build is never published;
  - `PUBLISH.md` mirroring the template's checklist: clean and synced `develop`, green CI, docs match code, `npm pack --dry-run` contents, unused version, `npm whoami`, fast-forward `master` to `develop`, publish from `master`, prerelease dist-tags, 2FA (`--otp`, generated fresh because `prepublishOnly` runs first), and first-publish registry 404 caching.
- **C9:** this repo's root `tsshowcase.config.json`: `scope: ['src']` and `strict: ['check', 'warnings']` (A4, §8); `fixtures/` needs no `ignore` (outside every package, A4; replaces D107's `ignore: ['/fixtures/']`).
- **C10:** the showcase smoke-render test (§10c).

## Amendment A1: JSON schemas with tsschema sidecars (D58, D62), re-approved

Raised during C2 planning. Our JSON Schemas (the config schemas in the CLI package (D66), our modules' options schemas) are plain `*.schema.json` files typed through committed `@ofzza/tsschema` sidecars (`*.d.json.ts`). C1 wires the tooling so later chunks only add schema files:

- **Dependencies:** `@ofzza/tsschema` as a root devDependency (for the `tsschema` command). Packages that publish schema-derived types (`-api` for `OptionsOf` from C2, the CLI for the config types from C6) add it as a regular dependency themselves.
- **TypeScript:** `allowArbitraryExtensions` in `tsconfig.base.json` (so JSON imports pick up their sidecars); each package's `tsconfig.build.json` `include` adds `src/**/*.json` (with a sidecar present, `tsc` no longer copies the JSON on its own).
- **Scripts:** `schemas` generates sidecars for `packages/*/src/**/*.schema.json`; `ci:schemas` runs it with `--check`, failing when a committed sidecar is stale. `build` = `schemas` → `tsc -b` → `scripts/copy-sidecars.mjs`. The `schemas` script must succeed when there are no schema files yet (C1 has none).
- **`scripts/copy-sidecars.mjs`:** copies each `src/**/*.d.json.ts` to the matching path under `dist/`. **Why:** `tsc` never copies declaration files from source, and without the sidecar a consumer's types inferred from the JSON silently become `never` (verified by experiment with TS 6.0.3 during C2 planning). With the sidecar in `dist`, consumers need no tsconfig changes.
- **Formatting and linting:** sidecars are generated and committed; `.prettierignore` and the ESLint ignores list `**/*.d.json.ts`.
- **Docs:** AGENTS.md gains the gotcha ("sidecars don't reach `dist` on their own") and a trigger-table row: "changed a `*.schema.json`" → run `npm run schemas`, commit the sidecar, update the README documenting it.
- **Acceptance (added):** 11. `npm run ci:schemas` passes with zero schema files; a throwaway `*.schema.json` gets a sidecar from `npm run schemas`, `ci:schemas` fails after editing the JSON without regenerating, and `npm run build` puts the sidecar into `dist`. Reverted afterwards.

## Amendment A2: versioning from `0.1.0` (D80), re-approved

Raised during C2 planning:

- Every package skeleton starts at `version: 0.1.0`, with internal dependency ranges `^0.1.0`. The first publish (C14) is `0.1.0`.
- **No changesets before the first publish.** The PR changeset check is wired in C1 but disabled (e.g. behind a `if: false` / an env switch, documented in AGENTS.md); C14 switches it on after the first publish.
- AGENTS.md gotchas gain: "before 1.0, a breaking change uses a **minor** changeset; a major changeset on a `0.x` package jumps to `1.0.0`".
- Acceptance criterion 7 (`npx changeset status` works) is unchanged.

## Amendment A3: the `-test` package and fixture ignore rules (D104, D107, D109), re-approved

Raised during C4 planning (`04a-c4-plan-review.md` M2, M4, N15):

- **Package:** a private `packages/tsshowcase-test/` (`@ofzza/tsshowcase-test`, `private: true`, never published) built from the §2 skeleton; it holds `copyFixture()` and `fixturePath()` (specified in C4 §11.1), is a `devDependency` only (no project reference), and skips itself from this repo's dogfood `check` with its own `tsshowcase.config.json` (`{ "skip": true }`, D59). §1, §2 and §6 (changesets `privatePackages`) updated in place.
- **Fixtures (§5):** their own ignore rules go in `.tsshowcaseignore` files (D109); fixture content this repo's `.gitignore` would catch is re-included with `!` rules; `fixtures/broken-*/` is excluded from Prettier; gotcha 2 annotated (C4 uses no git and no parent `.gitignore`); the C9 self-check uses config `ignore` (D107; superseded by A4: `fixtures/` lies outside every package).

## Amendment A4: the representation redesign (D160–D200), approved 2026-10-20

Plan 3 of the redesign's application (`08b-representation-redesign.md` §12, `08e` E7), after C2 amendment B9 (`02b-c2-b9-review.md`, which also lists the C1 checklist); its review and the user's decisions are in `01e-c1-a4-review.md`:

- **§8 Dogfooding:** units (a directory with an `index.ts` barrel, D36) are replaced by the representation rules: one exporting file per concept, one README per area documenting every exporting file in it, tests and showcases named after the file they cover, areas as plain directories (menu levels through `.tsshowcase`), collections for multi-file concepts, `src/index.ts` re-exporting declaring files; no per-concept barrels, no `_`-prefixed internal directories; JSON schemas imported by public code are exporting files; the dogfood config's `scope: ['src']`.
- **§5, "Recorded for later chunks":** `fixtures/` lies outside every package, so the dogfood check needs no `ignore: ['/fixtures/']`.
- **§9:** the layout wording and the glossary seed use the representation terms.
- **§10:** trigger rows for exporting files, area directories, directory configs, `kinds`/`exempt` entries, showcasing components and module kinds; one place per fact (the directory README or `<Concept>.md`); the `check` row (representation) and the smoke render per showcasing component (D195).
- **§1, §2, §4:** `-lint` noted in the layout (C3, D88); the skeleton comment ("no exported symbols"); an ESLint rule against `export *` in entry files; `typecheck.include` adds `*.test-d.{ts,tsx}`; extra Vitest projects per package (`unit:<pkg>:<part>`); the `debug` project opt-in through `TSS_VITEST_DEBUG=1` and a `debug` script; acceptance criterion 4 extended.
- **§8 (from the review):** directory READMEs per directory with exporting files; entry files re-export by name, carry `source` conditions, nothing exporting directly in `src/`; kinds by value (D200); local declarations for public API from dependencies; JSON exporting only when re-exported; `.tsshowcase.jsonc` only, no `directories` entries; the root config's `scope` and `strict`; root documents as home docs.
- Unchanged: the tooling, CI, fixtures convention (apart from the ignore), versioning, and the C1 deliverables themselves (C1 packages still export nothing).

## Amendment A5: the TS template as the source (D203), approved 2026-10-21

Raised by the user after plan 5 of the redesign's application: make sure the template stays the source and isn't forgotten, so the user's projects share one lineage. The user's direction (`01g-c1-a5-review.md`): no `project` skill; copy by hand what suits, merge and consolidate interactively with the user, and log the import afterwards.

- **Baseline:** the template is "the source"; the import step and the import log in `TEMPLATE.md` (above; TP2).
- **§1, §9, §10a:** `TEMPLATE.md` in the layout; AGENTS "Project" links to it; a trigger row for template changes.
- **AGENTS:** D203; D52's status.
- Unchanged: everything else in this plan; the adopt-or-keep decisions for individual differences (debug variable, test globs, ESLint setup, tsconfig details, …; listed in `01f-c1-a5-template-notes.md`) are made with the user during the import, and the plan text is aligned then.

## Resolved questions

- **Q1 Node floor → D49:** `engines.node: ">=22.12"` for every package, matching Vitest 5. This revises D17, since Node 20 has been end of life since 2026-04-30 and Vitest 5 / changesets need 22. The dev toolchain uses Node 24 (`.nvmrc`).
- **Q2 TypeScript → D50:** this repo pins TypeScript **6.0.x**, because `typescript-eslint` supports TypeScript below 6.1 only. Move to 7.x once it does. C8's analysis uses the showcased repo's own TypeScript (separate concern, settled in C8).
- **Q3 Licence → D51:** MIT, `author: ofzza`, for the root and every package.
- **Q4 Skeleton:** an empty entry (`export {};`) plus one `index.test.ts` wiring test per package.
- **Template review → D52–D55:** the template is the baseline (D52); the docs split adds CONTRIBUTE.md with a glossary, and PUBLISH.md comes in C14 (D53); the CI matrix is `[22, 24]` from C1 (D54); Prettier checks the whole repo (D55). `engines >=22.12` stays on every package, `-api` included (D49).
- **§10 → D56:** the extended trigger table, one place per fact, and the automated checks (`ci:readme-samples`, `ci:links` and the PR changeset check in C1; the self-check in C9; the showcase smoke render in C10).
- **Package directories → D57:** full npm names without the `@ofzza/` scope (e.g. `packages/tsshowcase-module-react-default/`).
