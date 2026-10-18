# Chunk plan C1: Monorepo foundation

> Status: **APPROVED** (2026-10-17), including amendment A1 (JSON schemas with tsschema sidecars, D62), re-approved the same day; see the end of this file. Amendment A2 (versioning from `0.1.0`, D80) re-approved 2026-10-18; see the end of this file. Editorial corrections: A1's wording updated after D66 moved the config schemas to the CLI package (C2 review F42); the glossary seed term "region class" replaced by "shell (shadow root)" after D89 (C3 planning). No change in scope. Amendment A3 (from C4 planning: the private `-test` package, fixture ignore rules; D104, D109, D107) re-approved 2026-10-18 with the C4 plan; see the end of this file. Note: C3 later adds an eighth package, `-lint` (D88), and C4 a private ninth, `-test` (D104); "7 packages" below refers to C1's own scope (01b K29). Decisions referenced as `D<n>` are in AGENTS.md §3.

## Goal

An empty but fully wired npm-workspaces monorepo. Every later chunk drops code into a package that already builds, lints, type-checks, tests and runs in CI.

C1 adds **no product code**: no contracts, CLI commands, themes or analysis. Those belong to C2 onwards.

**Baseline (D52):** the user's TypeScript project template at `~/projects/ofzza/.templates/ts` is the reference for tooling, conventions and docs. Everything below mirrors it, adapted from a single package to a monorepo. Deviations are called out explicitly.

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
  tsshowcase-test/                        @ofzza/tsshowcase-test             (private, added by C4: test helpers, A3)
scripts/
  fixtures.mjs                 install / clean the fixtures' own dependencies
  readme-samples.mjs           type-checks the ts/tsx code samples in every README (§10)
.editorconfig  .gitignore  .nvmrc  .prettierrc  .prettierignore
eslint.config.js  tsconfig.base.json  tsconfig.build.json  tsconfig.json  vitest.config.ts
package.json  package-lock.json  LICENSE  README.md  CONTRIBUTE.md
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
    index.ts             entry: file header + `export {};` (no exported units yet)
    index.test.ts        wiring test: the entry imports cleanly
```

**`package.json` fields:**

- `name`, `version: 0.1.0` (A2, D80), `description`, `keywords`, `license`, `author`;
- `repository` with `directory: packages/<pkg>`, plus `homepage` and `bugs`;
- `type: module`, `engines.node: ">=22.12"` (D49; a deliberate _runtime_ floor, see §11), `sideEffects: false` (except where a later chunk needs CSS side effects);
- `exports`: `{ ".": { "source": "./src/index.ts", "types": "./dist/index.d.ts", "default": "./dist/index.js" }, "./package.json": "./package.json" }`; `main` and `types` mirror it for older tooling (as the template);
- `files: ["dist"]`, `publishConfig.access: public` (the CLI package also lists its two config schema copies at the package root, made by its own `build` script: C6 §5, wording-only note from C6 planning);
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
  - `eslint-plugin-react-hooks` is scoped to `base-react` and `module-react-default` and added in C10, when there is React code to lint.
  - Stylelint and the D46 styling rules arrive in C3.
- **Vitest (the template's setup, multiplied by package):**
  - one root `vitest.config.ts` with `reporters: ['tree']` and `test.projects`.
  - **`unit:<pkg>`**, one per package, named after its directory (e.g. `unit:tsshowcase-api`):
    - runs `src/**/*.{test,unit.test,spec,unit.spec}.{ts,tsx}`;
    - **type-checks the same files** through `tsc` with the package's `tsconfig.json` (`typecheck.enabled`). Type assertions (`expectTypeOf`, `@ts-expect-error`) and type-only test files (`*.test-d.ts`, D38) therefore fail as tests;
    - `typecheck.spawnTimeout: 10000` (the template's Vitest 5 race workaround);
    - like the template, every test file is reported twice, once per pool (gotcha documented);
    - Node environment by default (C10 switches the React packages to jsdom).
  - **`debug`**, one project across all packages: runtime only, no type checking. It also matches `*.debug.test.ts` scratch tests, which are gitignored.
  - Workspace imports resolve via the `source` condition, so tests never need a build.
- **`.vscode/launch.json`:** the template's three configs ("Debug ALL Test Files", "Debug UNIT Test Files" → `--project "unit:*"`, "Debug Current Test File" → `--project debug`).
- **Root scripts** (with `npm-run-all2`, as the template):
  - `clean`, `build` (`prebuild: clean`), `dev` (`tsc -b --watch`), `lint` (ESLint), `format` / `format:check` (Prettier), `typecheck` (root files), `test` (`npm-run-all test:*`; `test:unit` → `vitest run --project "unit:*"`);
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
  2. **Git state leaks in.** (A3: since C4 reads no `.gitignore` above the scan root and uses no git at all (D103, D110), this no longer affects C4; it stays relevant to any later chunk that touches git.) Scanning and git info (C4) would find this repo's `.git` and root `.gitignore`. That makes "not a git repo", a GitLab/Bitbucket/missing remote, a detached HEAD, a shallow clone and a missing `origin/HEAD` untestable. Nested `.git` directories can't be committed either.
  3. **Tests that write files.** `init` writes stub files (C8/C9). `build` writes an output dir (C11), and its output guard needs a non-empty directory tsshowcase didn't create. These tests must not change the committed fixtures, or collide with each other when run in parallel.
- **The fix: a test-only `copyFixture(name, options) → { dir, cleanup }` helper, specified and implemented in C4** (the first chunk whose tests need it):
  - copies `fixtures/<name>/` into a fresh temp directory under `os.tmpdir()`. Nothing above it has a `node_modules` or `.git`, so a missing dependency really is missing;
  - skips `node_modules` when copying. Optionally links the fixture's already-installed `node_modules` into the copy, so nothing is reinstalled per test;
  - optional `git` settings: `git init` plus a commit, then set up the state the test needs (remote URL or none, branch name, detached HEAD, no `origin/HEAD`, …);
  - `cleanup()` removes the temp directory (called from `afterEach`).
  - Tests that only read a fixture and are unaffected by the three cases above may use `fixtures/<name>/` in place.
  - It lives in the private `-test` package (C4, D104; A3).
- C1 ships only `fixtures/README.md` describing this. The first fixtures arrive with C4.
- When `tsshowcase check` dogfoods this repo (C9), `fixtures/` is ignored via this repo's config `ignore: ['/fixtures/']` (D107). Recorded for C9.
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

- C1 packages export **no units** (`export {};`), so they satisfy the convention trivially. The convention is documented now and binds every later chunk.
- **This repo's unit layout** (it matches what C8's base rules will enforce):
  - `packages/<pkg>/src/<Unit>/` is a unit directory: `index.ts` (re-exports the unit's public symbols), `README.md`, `<Unit>.test.ts(x)`, and `<Unit>.showcase.tsx` where D37 requires one;
  - optional grouping directories (`src/<group>/<Unit>/`, D36) may be used where a package grows large;
  - each package's `src/index.ts` re-exports its units via `export * from './<Unit>/index.js';`;
  - `*.showcase.tsx` and `*.test.ts(x)` are excluded from `tsconfig.build.json`, so they never reach `dist`.

### 9. Documentation (D53)

The template's three-way split, adapted:

- **`AGENTS.md`** (agents). §1–§6 keep their role as the process and decision log. It gains the template's agent sections:
  - **Keeping documentation and tests current:** the trigger table, extended for this repo (§10);
  - **Project:** what tsshowcase is, licence, ESM only, `files` allowlist, "toolchain last verified against" line;
  - **Branching** and **Commits** (the template's text: commit to the current branch, never to `master` or a detached HEAD, propose feature branches, `develop` trunk / `master` stable, no AI attribution);
  - **Layout:** the tree above, what each package is for, and the unit convention (§8);
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
- **`CONTRIBUTE.md`** (contributors): reporting issues and the contribution workflow (linking the ofzza onboarding guides, as the template), branching, the project structure, and a **glossary**. The glossary is seeded now with the terms already in use: unit, showcase, showcased repo, rendering module, base package, theme, theme catalogue, shell (shadow root), test runner, fixture, interface version.
- **`README.md`** (users): a short stub now (what tsshowcase is, "under construction", Development and Contributing sections). C13 writes the real one.
- **Package READMEs:** the template's structure (§2).
- **`PUBLISH.md`:** arrives with C14, mirroring the template's checklist (see "Recorded for later chunks").
- **`LICENSE`:** MIT, ofzza (D51).

### 10. Keeping docs, tests and showcases current at scale (D56)

The template relies on a trigger table and discipline. This repo will have dozens of unit READMEs, tests and showcases across 7 packages, so the plan is to **keep the trigger table** and **back it with automation wherever "out of date" can be detected mechanically**.

**a) The trigger table, extended.** The template's rows are kept, and these are added:

| A change that …                                         | … requires updating                                                                                                                                          |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| adds a unit                                             | its directory, `index.ts`, `README.md`, test file, showcase where D37 requires one; the package's `src/index.ts` re-export; the package README's export list |
| changes a unit's signature, props, options or semantics | the unit's README, tests (and their purpose comments) and showcase; every README sample using it                                                             |
| removes or renames a unit or an export                  | the package entry, the package README's export list, every README linking to it, and its showcase                                                            |
| changes an `-api` contract                              | the interface changelog (D13: additive within a major), the authoring docs, and every in-repo implementer (built-in module, theme, runner)                   |
| changes a theme catalogue class or variable (from C3)   | the catalogue docs, the default themes, and the module/runtime code using it                                                                                 |
| changes a CLI flag, config key or output format         | the CLI reference in the root README and the config schema docs                                                                                              |
| adds a package                                          | the layout and packages list in AGENTS.md, the root `tsconfig.build.json`, Vitest projects, the dependency table, CONTRIBUTE.md's structure                  |
| changes a fixture                                       | the tests using it and `fixtures/README.md`                                                                                                                  |
| changes a package's public behaviour at all             | a changeset (`npm run changeset`)                                                                                                                            |

**b) Single source per fact.** Each fact is written once and linked from elsewhere, so there are fewer places to update:

- a package's export list lives in that package's README only (AGENTS.md links to it, and doesn't repeat it as the template does);
- a unit's documentation lives in its own README only. Package READMEs link to it rather than repeating it.

**c) Automated checks, added to `npm run ci`:**

| Check                                                                                                                                                                                                                                                                                                 | Catches                                                                     | Chunk  |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------ |
| `ci:readme-samples`: `scripts/readme-samples.mjs` extracts every ` ```ts `/` ```tsx ` block from every `README.md`, and type-checks them against the workspace (imports of `@ofzza/tsshowcase-*` resolve to source via the `source` condition). A block can opt out with a `nocheck` fence attribute. | README samples drifting from the API (automates the template's manual rule) | **C1** |
| `ci:links`: `remark-validate-links` over every Markdown file (offline; relative files and heading anchors).                                                                                                                                                                                           | Links broken by a rename, move or removed heading                           | **C1** |
| Unit completeness (own dir, README, tests, showcase per kind): `tsshowcase check` on this repo (D41)                                                                                                                                                                                                  | A unit missing its README, tests or showcase                                | C9     |
| Showcase smoke render: every `*.showcase.tsx` renders without throwing (stdblox's `showcases.test.tsx` pattern)                                                                                                                                                                                       | Showcases broken by an API change                                           | C10    |
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
4. A deliberately broken type assertion in a test fails `npm test` (the template's sanity check). Reverted afterwards.
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
- **C9:** ignore `fixtures/` in this repo's own `tsshowcase check` (config `ignore: ['/fixtures/']`, D107).
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
- **Fixtures (§5):** their own ignore rules go in `.tsshowcaseignore` files (D109); fixture content this repo's `.gitignore` would catch is re-included with `!` rules; `fixtures/broken-*/` is excluded from Prettier; gotcha 2 annotated (C4 uses no git and no parent `.gitignore`); the C9 self-check uses config `ignore` (D107).

## Resolved questions

- **Q1 Node floor → D49:** `engines.node: ">=22.12"` for every package, matching Vitest 5. This revises D17, since Node 20 has been end of life since 2026-04-30 and Vitest 5 / changesets need 22. The dev toolchain uses Node 24 (`.nvmrc`).
- **Q2 TypeScript → D50:** this repo pins TypeScript **6.0.x**, because `typescript-eslint` supports TypeScript below 6.1 only. Move to 7.x once it does. C8's analysis uses the showcased repo's own TypeScript (separate concern, settled in C8).
- **Q3 Licence → D51:** MIT, `author: ofzza`, for the root and every package.
- **Q4 Skeleton:** an empty entry (`export {};`) plus one `index.test.ts` wiring test per package.
- **Template review → D52–D55:** the template is the baseline (D52); the docs split adds CONTRIBUTE.md with a glossary, and PUBLISH.md comes in C14 (D53); the CI matrix is `[22, 24]` from C1 (D54); Prettier checks the whole repo (D55). `engines >=22.12` stays on every package, `-api` included (D49).
- **§10 → D56:** the extended trigger table, one place per fact, and the automated checks (`ci:readme-samples`, `ci:links` and the PR changeset check in C1; the self-check in C9; the showcase smoke render in C10).
- **Package directories → D57:** full npm names without the `@ofzza/` scope (e.g. `packages/tsshowcase-module-react-default/`).
