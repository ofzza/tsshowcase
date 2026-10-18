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
| 2 | Super-plan: partition the work into chunks, order them, mark which can be planned in parallel and which sequentially; iterate until approved | ✅ **approved**: `.plans/01-super-plan.md`, with the review triage and final consistency review in `.plans/01a-super-plan-review.md`; revisions R1 (with C2) and R2 (with C3) re-approved |
| 3 | Chunk plans: one at a time, in super-plan order; iterate and approve each | 🔄 in progress: C1, C2, C3 ✅ approved (super-plan R1, R2 re-approved); 🔜 next: **wave P3** (C4, C5, C6, C7, C15, any order) |
| 4 | Implementation: one chunk at a time; explain each and wait for review before the next | ⏳ |

### How to resume (fresh session)

1. Read this file, then `.plans/01-super-plan.md` (chunk definitions, dependencies, planning and implementation order), and `.plans/01a-super-plan-review.md` (the **"Deferred to chunk plans"** list: open points to raise when planning each chunk). The per-chunk review triages (`.plans/02a-c2-plan-review.md`, `.plans/03a-c3-plan-review.md`, `.plans/03b-c3-plan-review.md`, …) record what fresh-eyes reviews found and how each finding was resolved.
2. Find the next step in the stage table above.
3. **Chunk planning loop** (stage 3):
   - draft `.plans/chunk-NN-<slug>.md`;
   - raise the open points for that chunk and ask questions (any number of rounds);
   - iterate until the user explicitly approves;
   - record new decisions here as `D<n>`;
   - mark the chunk approved in the chunk-plan tracker below;
   - commit.
4. **Implementation loop** (stage 4, only after *all* chunk plans are approved):
   - implement one chunk;
   - explain it to the user and wait for their review before moving on;
   - commit.
5. Backtracking to any earlier plan is always allowed. Record it here, update the affected plans, and get re-approval.
   - **Super-plan drift:** when a chunk-plan decision changes the super-plan, update the super-plan in the same step, adding a revision note (`R<n>`) listing the decisions that caused it. The super-plan is re-approved together with that chunk plan.
   - **Approved chunk plans:** edits that only correct wording are noted in the plan's status line and need no re-approval; any change in scope needs re-approval.
6. `REQUIREMENT.md` is the user's verbatim original request, and the source of truth for the process expectations.

### Chunk-plan tracker

| Chunk | Plan file | Status |
|-------|-----------|--------|
| C1 Monorepo foundation | `.plans/chunk-01-monorepo-foundation.md` | ✅ approved (incl. amendments A1, D62, and A2, D80) |
| C2 Plugin contracts | `.plans/chunk-02-plugin-contracts.md` (review: `02a-c2-plan-review.md`) | ✅ approved, incl. amendment B1 (closed) |
| C3 Theme system | `.plans/chunk-03-theme-system.md` (reviews: `03a-c3-plan-review.md`, `03b-c3-plan-review.md`) | ✅ approved (round 6) |
| C4 Repo scanning | – | 🔜 next (wave P3) |
| C5 Markdown pipeline | – | 🔜 next (wave P3) |
| C6 Plugin loading + config | – | 🔜 next (wave P3) |
| C7 Vitest runner | – | 🔜 next (wave P3) |
| C15 Vite helper | – | 🔜 next (wave P3) |
| C8 React base (Node) | – | ⏳ |
| C9 CLI check + init | – | ⏳ |
| C10 React runtime + default module | – | ⏳ |
| C11 CLI build + dev | – | ⏳ |
| C12 E2E | – | ⏳ |
| C13 Docs | – | ⏳ |
| C14 Release | – | ⏳ |

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
| D7 | Themes: **several per build, with a runtime switcher** | The default follows `prefers-color-scheme`. The theme system defines a switcher class that modules render |
| D8 | **Failing tests never block `build`** | Failures are shown per unit in the showcase. A `--fail-on-test-failure` flag gives strict mode (CI) |
| D9 | **How module/theme references resolve, in order:** (1) the name of a module/theme bundled with the CLI, (2) an installed npm package name, (3) a filesystem path | JS and TS are both allowed. TS is loaded at runtime with a loader (e.g. jiti), so custom module authors need no build step |
| D10 | **Adopt the stdblox code conventions** | Prettier `printWidth: 160`; British spelling; file header comments; JSDoc everywhere; `_` prefix for internals; `// #region` blocks; a purpose comment opening every `describe` |
| D11 | **Validation uses in-repo fixtures only** | Do not touch stdblox-react for now |
| D12 | **Licence file:** found case-insensitively as `LICEN[CS]E` with an optional `.md`/`.txt` extension, for each package in workspaces | Confirmed |
| D13 | **Contract versioning uses a semver range declared in code** (from `1.0.0`; before it, minors may break: D80) | Every module, theme and test runner declares `static readonly apiVersion = '^1.2'`.<br>• **The interface has its own version**, independent of the CLI package version, plus an **interface changelog** ("added in 1.3: …"). It is exported as `IMPLEMENTED_API_VERSION`.<br>• The CLI checks `semver.satisfies(IMPLEMENTED_API_VERSION, apiVersion)` at load time and fails with a clear message.<br>• **Within a major version, changes are additive only.** Methods added after 1.0 are optional and feature-detected; modules ignore fields they don't know.<br>• The types package is versioned in step with the interface.<br>• **Rejected:** requiring a `package.json` with a peerDependency on the interface package for path-loaded modules |
| D14 | **This repo is an npm workspaces monorepo** (plus `-lint`, D88) | Separate packages (full list: super-plan package table): `-api`, the CLI, themes, the test runner, `-base-vite`, `-base-react` and the React module. The CLI depends on the built-in packages so that lookup by name works (D9) |
| D15 | **Naming:** `@ofzza/tsshowcase` (the CLI, binary `tsshowcase`), with siblings `@ofzza/tsshowcase-<kind>-<name>` | See D27 for the full scheme |
| D16 | **v1 scope adds:** a `dev` action (watch + HMR; module-owned via D43); an `init`/scaffold action (generate the stubs a module requires); **workspace (monorepo) showcased repos**; a machine-readable check report (`--format json`, non-zero exit on errors) | **Interface consequences:**<br>• the module needs a scaffold method;<br>• the input model must be multi-package from day one (a single-package repo = one package);<br>• `dev` needs a test strategy |
| D17 | ~~**Node >= 20**~~, **ESM only** | The Node floor is **revised by D49** (>= 22.12) |
| D18 | **In `dev`, tests run in watch mode** | Results are pushed live to the open showcase through the module's dev server (D42, D43), so the test-runner interface must support watch and streamed updates |
| D19 | **Showcase hierarchy for workspaces:** repo README (home) → each package (its README and licence) → that package's units | The package level collapses for a single-package repo |
| D20 | **Layout variants are separate rendering modules** (e.g. a React per-page module vs a React single-page module) | They share a common React base (resolved by D26/D48: the `-base-react` package with a base module class) |
| D21 | **React verification:** base rules by default, opt-in rule presets | Base rules: own directory, README, test, showcase, exported from the entry (test and showcase requirements refined by D37/D38). The headless pattern (`useX`/`renderX`/`X`, from stdblox) is an opt-in preset named `headless` (D47) |
| D22 | **Only the React default module (`react-default`, D27) ships in v1** | The shared base (D26) makes further React modules cheap to add |
| D23 | **The CLI pre-renders markdown to HTML** | Output carries the documented theme classes, heading ids and rewritten links. Modules receive both the raw markdown and the HTML |
| D24 | **How repo-file links are rewritten:** the host (GitHub/GitLab/Bitbucket) is detected from `package.json` `repository` and the default branch from git; a configurable URL template overrides both | Links to unit READMEs become showcase routes |
| D25 | **Interactive playground is out of v1 scope** | No provision for it in the v1 contract (decided during C2 planning): too many unknowns, and if it ever comes it is the module's business how it relates to a showcase. The additive-only rule (D13) is all the room it needs |
| D26 | **The React base ships as a separate package from v1:** `@ofzza/tsshowcase-base-react` | **Base:** layout-agnostic parts (analysis, rules, scaffolding, test-result mapping, the React wrapper around `-base-vite` per D48) and runtime building blocks. It is not a loadable module.<br>**Default module:** layout only (shell, nav, routes, page composition). The user chose this over the recommended internal split |
| D27 | **Naming scheme `@ofzza/tsshowcase-<kind>-<name>` (kinds: `module`, `base`, `theme`, `testrunner`, plus the nameless `-lint` (D88); the test-runner kind was renamed from `runner` to be explicit):**<br>• `-module-react-default` (built-in name `react-default`; renames "react-pages");<br>• `-theme-default` (`default-light`, `default-dark`);<br>• `-testrunner-vitest` (`vitest`);<br>• `-base-react`;<br>• `-api` | "single-page vs pages" was only an example of why several modules might exist, not a v1 requirement |
| D28 | **The theme catalogue lives in `-api` as importable typed constants** | Every class name and CSS variable name. Modules import the constants and never hardcode strings. C3 designs the hierarchy and implements the default themes |
| D29 | **Implementation order builds the "inputs" first** (`check`/`init` slice: C1→C2→C4→C6→C8→C9) | Lock down the inputs (scanning, analysis, data model) before moulding them into a themed showcase app |
| D30 | **The former C12 is split** into C12 (E2E acceptance), C13 (docs) and C14 (release) | Release scope settled in D33 |
| D31 | **Packages are versioned independently, with changesets** (starting at `0.1.0`: D80) | Clarifies D13: **the `-api` package version *is* the interface version** (`IMPLEMENTED_API_VERSION` is derived from it), and changesets bumps it like any other package. "Independent" means the other packages' versions don't follow it |
| D32 | **Basic CI from C1** (lint, type-check, test, build on push/PR) | C14 extends it |
| D33 | **C14 scope:** CI extensions (Node matrix, E2E), pack-and-install smoke test, a **manual publish shell script** (dependency order, dry-run; no automated publish workflow for now), and a dogfood docs site on GitHub Pages | |
| D34 | **Test ownership: the module's `runTests()` calls a runner library.** The runner is chosen through a **module option** (`moduleOptions.testRunner`, e.g. `'vitest'`; auto-detected if unset, D42) | The CLI resolves runner references (built-in name → npm → path, D9) on the module's behalf, but never calls a runner itself. Refines D3 |
| D35 | **Routes: the module exposes `resolveRoute(target, heading?)`** (signature revised by D67) | `target` is home, a package, a group (D78) or a unit. The CLI's markdown renderer (D23) calls it to rewrite README file links into final in-app URLs. Links to non-README repo files go to the VCS host (D24) |
| D36 | **A unit is a directory exporting one or more public symbols. Nesting is allowed** (a unit may contain sub-units) | Intermediate non-unit directories become menu levels |
| D37 | **Requirement change: README and tests are mandatory for every unit; showcases are mandatory per kind, as the module decides** | `react-default`: required for components and hooks; optional for functions, classes and constants; not applicable to type-only units. Severity can be overridden via rule IDs (D21). The showcase path is optional in the data model, and layouts must handle its absence. Pure-TS libraries therefore need no React unless they opt into showcases |
| D38 | **Type-only units require only a README** | Tests are optional; type tests (`expectTypeOf`, `*.test-d.ts`) are shown if present. Exception to D37's "tests mandatory" |
| D39 | **Syntax highlighting is kept, class-based only** | Done at build time in the CLI markdown pipeline (e.g. Shiki with a CSS-variables theme, or Prism/highlight.js classes). Token colours are catalogue variables; no inline colours |
| D40 | **`verify` receives the file list plus lazy, cached `read(file)` accessors** instead of eager contents | Requirement change: semantically equivalent, but only files the module needs are read, and the CLI can track reads for `dev` invalidation |
| D41 | **This repo dogfoods its own convention from C1** | Every exported unit in this repo's packages has its own dir, README and tests, with showcases per D37 (the `-base-react` components need them). `tsshowcase check` runs on this repo in CI once C9 lands. C14 only adds the GitHub Pages deploy |
| D42 | **Test flow and runner loading** (refines D3/D34; loading refined by D74, re-analysis by D75, `ctx.onTestUpdate` renamed `reportTestUpdate` in C2):<br>• the CLI calls `module.runTests(ctx)`;<br>• the module calls `ctx.loadTestRunner(ref?)` and gets back a resolved, version-checked runner;<br>• the runner's `run`/`watch` drive the **showcased repo's own** test tool and return a raw hierarchy (file → suite → test);<br>• the **module maps results to units** and returns them;<br>• in `dev`, `watch` updates go module → `ctx.reportTestUpdate` → CLI → the module's dev server (`DevContext.onTestUpdate` subscription) → browser | **Auto-detection:** runners implement `detect(repo)`.<br>• An explicit `ref` (config `moduleOptions.testRunner`, or a module default) wins.<br>• Otherwise the CLI calls `detect` on each candidate runner (built-ins plus extra candidates from the config). Exactly one match: use it and log it. Several: error with a hint. None: tests are unavailable (warning).<br>• Modules may restrict the candidates.<br>• The runner never throws on failing tests (D8). If the runner itself fails, the affected units show "tests unavailable" |
| D43 | **Hybrid build ownership (supersedes D4):** the contract says the **module owns `build(ctx)` and `dev(ctx)`**. The CLI assembles the data (packages, units, README HTML, tests), resolves the themes and the output dir, watches files and re-analyses in `dev`, and pushes updates to the module (e.g. `ctx.onDataUpdate`). | The **Vite pipeline ships as a reusable helper package, `@ofzza/tsshowcase-base-vite`**. It handles data embedding (virtual module), theme CSS inclusion and switcher setup, `base: './'`, the output dir, the dev server, HMR and the live channel. `react-default` uses it by passing its plugins and entry. Modules for other frameworks may bring their own toolchain, but must honour the data and theme inclusion rules defined in `-api`. Still applies: framework runtime packages resolve from the showcased repo (handled in `-base-react`). History: Claude first recommended module-owned, then CLI-owned (D4) after discussion, and finally the hybrid |
| D44 | ~~**Three-tier theming scope**~~ **superseded by D89** | Originally: variables inherited everywhere including showcases, catalogue classes usable in showcases, opt-in region classes theming bare elements, a cascade `@layer`. Replaced by the isolation model (D89): the shell lives in a shadow root and showcases are isolated light-DOM content; `tss-content` remains only as the README content class |
| D45 | **Prefix:** classes `tss-…`, CSS variables `--tss-…` | |
| D46 | **One styling policy, enforced in both CSS (stylelint) and inline styles/CSS-in-JS (ESLint).** Allowed: `var(--tss-…)` references, `calc()`/`color-mix()` built from them, and structural keywords (`display: flex`, `text-align: center`, `position`, `overflow`, `0`, `100%`, `auto`, `inherit`, …). Banned: literal design values (colours, lengths, font sizes, radii, shadows, durations, …) | Applies **only to tsshowcase's own module and theme packages**, never to user showcases. Escape hatch for genuinely dynamic values: a line-level disable comment that requires a written reason. This replaces the earlier idea of banning inline `style={}` entirely |
| D47 | **The opt-in React rule preset for the logic/presentation split is named `headless`** | `useX` holds the logic and prop getters, `renderX` is a pure presentation function, and `X` only wires them together. Config: `presets: ['headless']`. Not named after stdblox |
| D48 | **`-base-react` builds on `-base-vite`** | Its build/dev half is a React wrapper around `-base-vite` (plugin-react, React dedupe, tsconfig paths, runtime building blocks). Its check/init half (analysis, rules, scaffolding) has no bundler involvement. It exposes a **base module class** with every contract method implemented; layout modules (e.g. `react-default`) extend it and supply only their app entry and layout |
| D49 | **Node floor raised to `>=22.12`** for every package (revises D17) | Node 20 has been end of life since 2026-04-30. Vitest 5 needs `^22.12` and changesets `^22.11`. The user first chose to keep 20, then picked the single raised floor over a split (users ≥ 20.19, dev on 22+). This repo's dev toolchain uses Node 24 (`.nvmrc`). CI runs a `[22, 24]` matrix from C1 (D54); C14 adds 26 |
| D50 | **This repo's toolchain pins TypeScript 6.0.x** | TS 7.0 (the native port) is `latest`, but `typescript-eslint` supports TypeScript below 6.1 only. Move to 7.x once it does. Analysis of showcased repos (C8) uses *their* TypeScript |
| D51 | **Licence: MIT, author `ofzza`**, for the root and every package | Each package carries a copy of `LICENSE`, since npm only packs a package's own licence file |
| D52 | **The user's TS template (`~/projects/ofzza/.templates/ts`) is the baseline** for this repo's tooling, conventions and docs, adapted to a monorepo | Adopted from it: Vitest typecheck of tests (+ `spawnTimeout` workaround), the `debug` project and `*.debug.test.ts` scratch tests, `.vscode` extensions/launch configs, `.gitignore` conventions (`*.gitignore.*`, re-included `.vscode` files), code-style rules, the docs-and-tests trigger table, its gotchas, `@types/node` pinned to the floor major, and (for C14) its PUBLISH.md checklist and dual-resolution consumer smoke test. Deviation: `moduleResolution: nodenext` instead of `bundler` |
| D53 | **Docs split as in the template:** `AGENTS.md` (agents; also keeps the process and decision log), `CONTRIBUTE.md` (contributors, with a **glossary**), `README.md` (users). `PUBLISH.md` arrives with C14 | |
| D54 | **CI runs a Node `[22, 24]` matrix from C1** (refines D33) | C14 adds Node 26 |
| D55 | **Prettier checks the whole repo, Markdown included** | Deviation from the template, which checks `src` only. `REQUIREMENT.md` (verbatim) and generated changeset files are ignored |
| D56 | **Keeping docs, tests and showcases current at scale:** the template's trigger table, extended with unit, contract, catalogue, CLI, package, fixture and changeset rows; **one place per fact** (a package's export list lives only in its README, a unit's docs only in its own README); and **automated drift checks** | In C1: `ci:readme-samples` (type-checks every ts/tsx README sample; `nocheck` fence opt-out), `ci:links` (`remark-validate-links`: relative links and heading anchors), and a PR changeset check. Later: `tsshowcase check` on this repo (C9) and a showcase smoke render (C10). README prose and showcase content stay a manual review concern. Details in the C1 plan §10 |
| D57 | **Package directories use the full npm package name without the `@ofzza/` scope** | `packages/tsshowcase/` (CLI), `packages/tsshowcase-api/`, `packages/tsshowcase-module-react-default/`, …; Vitest projects are named after them (`unit:<dir>`) |
| D58 | **Schemas: JSON Schema (draft 2020-12) for module options and the config, validated with ajv in the CLI; TS types inferred with `@ofzza/tsschema`** | The schema is the source of truth: types are inferred, never hand-written. `-api` depends on tsschema (types only). Map-shaped fields infer as `unknown` until tsschema gains record/tuple inference (on its roadmap); acceptable until `1.0.0` (D80). Chosen over Standard Schema and zod. Storage form: D62 |
| D59 | **Per-package config files in workspaces** | A workspace package may have its own `tsshowcase.config.*`, merged over the root one, with a restricted schema (`exclude`, `rules`, `moduleOptions`); repo-wide keys there are an error. v1: package directories only; nested directory configs may come later, additively. Chosen over a root `packages` override map |
| D60 | **The `check --format json` report is a CLI detail, not part of `-api`** | `-api` holds only the internal plugin contracts. The report is external output: types exported from the CLI package, documented in the CLI README, with a `reportVersion` field. Shape specified in C9 |
| D61 | **A unit may have several showcases** (`showcases: ShowcaseEntry[]`, possibly empty) | Going from one to many later would be breaking |
| D62 | **Our JSON Schemas are plain `*.schema.json` files typed through committed tsschema sidecars** (`*.d.json.ts`, `tsschema generate --mode sidecar`), not `as const` literals | The schema is published as JSON (`$schema` completion, readable by any tool; docs in `description`). Costs: a `schemas` codegen script plus `ci:schemas` (`--check`), and a build step that copies sidecars into `dist`, because `tsc` doesn't and consumers' types would silently become `never` (verified by experiment). Applies to the config schemas and our own modules' options schemas; the contract (`optionsSchema: JsonSchema`) still accepts any value. Tooling is C1 amendment A1 |
| D63 | **Only `analyze` is a required module method; everything else (`verify`, `resolveRoute`, `runTests`, `build`, `dev`, `scaffold`, `dispose`) is optional and feature-detected per action** | A missing method fails its action with a clear "module X does not support `<action>`". Stubs are worse than that error (a stub `verify` silently passes CI). Modules that support no action are allowed; they are just useless. Claude proposed requiring `verify` too; the user overruled |
| D64 | **The CLI performs all writes; modules return what to write** | `build` returns the whole site as `BuildFile[]` (`path` relative to the site root, `contents` as string or bytes) and the CLI writes `outDir` (output guard, atomic swap, marker file, path validation, checkable inclusion rules; later possibly lint/format/minify). `BuildContext` has no `outDir`. Same as `scaffold`, which already returned files. Vite supports it natively (`write: false`, via `-base-vite`); disk-only toolchains build into `cacheDir` and return the result. `dev` writes nothing. A contract rule, not a sandbox. The user's proposal, extending the "CLI handles IO" principle from reads to writes |
| D65 | **Themes extend by composition; theme CSS is unscoped and the build scopes it** | No `extends` field in `ThemeDefinition`. A theme package depends on the base theme package (npm resolution, its own range) and lists the base's CSS files before its overrides. Required rule: theme CSS never references its own name or the theme attribute; the build scopes each theme's files under `[data-theme=<name>]` (mechanics in C3). Chosen over a CLI-resolved `extends` reference (resolution, cycles, version checks); that can still be added later, additively |
| D66 | **The config is a CLI detail, not part of `-api`** | Same reasoning as D60: it is user-facing and consumed only by the CLI (no plugin sees it; modules get only their validated options). The config schemas, inferred types and `defineConfig` live in `@ofzza/tsshowcase` (users: `import { defineConfig } from '@ofzza/tsshowcase'`, `$schema` → `node_modules/@ofzza/tsshowcase/config.schema.json`) and are documented in the CLI README. Implemented in C6. `defineConfig` stays as optional typing sugar for `.ts`/`.js` configs (convention); plain JSON configs remain fully supported. Module `optionsSchema` validation stays in the C2 contract |
| D67 | **`resolveRoute(target, ctx, heading?)`, and routes are stored in the data** (revises D35's signature) | `ctx` carries the analysis, so modules keep no state on `this`. The CLI calls `resolveRoute` for home, every package and every unit and stores `route` on `RepoNode`/`PackageNode`/`UnitNode`; the browser app reads routes from the data (one source of truth). From the C2 review (F2, F3, F35) |
| D68 | **`dev` freshness: `analyze` reports its `inputs`; dev updates are full snapshots with a `revision`** | `AnalysisResult.inputs` lists the files the analysis depended on; the CLI re-analyses when any of them or the structure changes (catches entry edits). `ShowcaseData.revision` and `TestResultsUpdate.revision` increase, so stale messages are dropped. Patches may come later, additively. From the C2 review (F4, F32, Q6) |
| D69 | **Per-package module options reach the module as `PackageInput.options`; the per-package config key is `skip`** | Root `moduleOptions` merged with the package's own, each result validated against `optionsSchema` (C6); `ModuleInit.options` stays the root value. `skip: true` drops the package from `RepoInput.packages` and its directories from the scan (renamed from `exclude` to avoid clashing with the root `exclude` patterns). From the C2 review (F6, F27) |
| D70 | **The CLI applies severity overrides and validates module output** | Overrides apply per issue, with the issue's package (derived from `dir`/`file` when unset) selecting per-package `rules`. `analyze` issues are always printed in `build`/`dev`. Invalid analysis (a unit dir missing, outside its package or `unitRoot`, duplicated, or in an unknown package) is a thrown error naming the module. From the C2 review (F12, F34, Q8) |
| D71 | **`Doc.format: 'markdown' \| 'text'`** | `.md` files are markdown; `LICENSE`/`.txt` files are text and render as escaped `<pre>` with a catalogue class (C5). `Doc.markdown` became `Doc.source`. From the C2 review (F13) |
| D72 | **A package that `analyze` leaves out is hidden from the showcase** | To keep a package visible without units, the module returns it with `units: []`. `check` still visits its directories. The user overruled Claude's recommendation (keep omitted packages visible) |
| D73 | **Link-to-target mapping (C5)** | A unit's README or directory → the unit; a package's README or directory → the package; the root README or repo root → home; anything else in the repo (incl. a unit's `index.ts`) → the VCS host (D24); external links untouched. From the C2 review (F22) |
| D74 | **`loadTestRunner(ref?, { candidates? })`** | `candidates` restricts auto-detection (D42's "modules may restrict"). Several matches **throw** (configuration error, hint to set `testRunner`); no match resolves `undefined` (units `unavailable`); an unresolvable or incompatible `ref` throws. From the C2 review (F7) |
| D75 | **`TestContext.onAnalysisUpdate(listener)`** | In watch mode the module receives every new `AnalysisResult` after a `dev` re-analysis, remaps test files to units and re-emits via `reportTestUpdate`; the runner keeps running. Chosen over restarting `runTests`. From the C2 review (F5) |
| D76 | **`check` is a prerequisite of `build` and `dev`** | Both run the full check (`analyze` + `verify`) and print the report. Check errors fail `build` by default (`--no-check` / config `check: false` opts out); warnings never fail; `dev` never fails on issues. A module without `verify` skips that pass with a note. Tests unchanged (D8). Claude first proposed `build` without `check`; the user's question turned it around (Q7, F33) |
| D77 | **How the showcase data is embedded in the output is the module's business** | Only inclusion rules 1–2 (static/relocatable, no leaks) constrain it; nothing outside the module's app reads it. `-base-vite` compiles it into the bundle. Resolves the super-plan's "how the data is embedded" bullet (F8) |
| D78 | **Group READMEs, and `resolveRoute` may return `undefined`** | The CLI sets `GroupNode.readme` when a group directory has a `README.md` (never required, never an issue). `RouteTarget` gains `group`; `undefined` = no in-app page, so `GroupNode.route` stays unset and links go to the VCS host. Home/package/unit targets must resolve. The user's proposal (F21) |
| D79 | **Module-defined CLI flags: `static optionFlags = { flag: 'option.path' }`** | Flags override options (root config < package config < flags, D6), parsed from the schema at the target path, listed in `--help`; collisions with CLI flags are a load-time error; accepted on every action. Replaces `ScaffoldContext.args`. The user's proposal (F30) |
| D80 | **Versioning starts at `0.1.0`** (refines D13, D31; replaces the `1.0.0` proposal) | Every package starts at `0.1.0`; the first publish (C14) is `0.1.0`; no changesets before it (PR check switched on after). In `0.x` a minor may break; additive-only (D13) from `1.0.0`, a deliberate later step. Plugins declare e.g. `apiVersion = '^0.1'`. Breaking changes before 1.0 use minor changesets (a major one jumps to `1.0.0`). C1 amendment A2. The user wants field-tested prereleases before committing to 1.0 (Q10, F36) |
| D81 | **No cascade `@layer` for theme or module CSS** (revises D44) | Isolation comes from the shadow root (D89), not the cascade. Inside the shell's shadow root, theme rules (zero-specificity `:where()` wrapper, D82) come first and module CSS after them. From C3 planning (Q2) |
| D82 | **The CLI scopes theme CSS into a shell stylesheet and a fonts stylesheet** | `scopeThemeCss` (PostCSS, CSS nesting) wraps each theme in `:where(.tss-shell[data-theme="<name>"])` (zero specificity), turns `:root` into `&`, adds a per-theme `color-scheme` block, groups shared files by topological merge (realpath identity), inlines `@import`, hoists `@keyframes`, moves `@font-face` into the fonts stylesheet (it doesn't work inside shadow roots), rejects `@layer`/`@property`/other at-rules and document selectors, rewrites `url()`s relative to the output. C11 writes both atomically to the CLI cache (`node_modules/.cache/tsshowcase/`) and passes `BuildContext.themeStylesheets: { shell, fonts }`; regenerated in place in `dev`. Implemented in C3, wired up in C11. From C3 planning (Q3) and review |
| D83 | **Catalogue reference docs are generated into the `-api` `catalogue/` README** | Audience: theme and module authors. The `-theme-default` README is for users theming their showcase. Moves the reference from `-theme-default` (super-plan wording). From C3 planning (Q5) |
| D84 | **States and variants use ARIA/data attributes, not modifier classes** | E.g. `aria-current="page"`, `aria-expanded`, `aria-pressed`, `data-status`, `data-kind`; attribute names and values are catalogue constants (`tssData`). Documented precisely per class (a "States and variants" section with CSS examples) so theme authors can follow it without reading module code. From C3 planning (Q6) |
| D85 | **Contrast: a review guideline in C3, smoke-tested in C12** (revised) | WCAG AA for the semantic pairs, checked by hand when the default themes change, and by a computed-contrast smoke test on the built output in C12's existing headless-browser run. Originally a reusable `checkThemeContrast` helper with an automated C3 test; dropped because C3 shouldn't introduce real-browser testing (review G10, H31) |
| D86 | **A theme may style anything inside the shell, within three rules** (revised by D89) | It may override any variable, style any catalogue class or bare element in the shell, and add its own variables, classes and selectors (inherited and overridable by composing themes). Rules: (1) no self-scoping or document selectors (`[data-theme]`, `:host`, `html`, `body`, compound `:root`); (2) its own names (classes, variables, keyframes, font families) use its own prefix, never `tss-`/`--tss-`; (3) modules only use catalogue names in markup, plus their own prefixed arrangement-only classes, which themes never style (amended per review G25). Documented with their purpose in "Building a theme that works with any module". Linted only on our own themes. The former "no bare elements outside a region" rule is obsolete under D89. From C3 planning (Q1) |
| D87 | ~~**Showcase styling opt-in with a `tss-plain` escape hatch (`@scope`)**~~ **superseded by D89** | Regions, `@scope (.tss-content) to (.tss-plain)` and its browser floor are gone: showcases are isolated from the shell instead |
| D88 | **`@ofzza/tsshowcase-lint`: a published package of lint rules, plugins and presets** | No code that runs linting: consumers run stylelint/ESLint with their own configs. Contents: the stylelint plugin (`design-values` with a structural allow-list, `known-variables`, `scoping`, `own-prefix`), the ESLint plugin (`style-values`, `known-classes`), presets `stylelintConfigs.theme`, `stylelintConfigs.module`, `eslintConfigs.module` (no rules for runners). Enforced on our own packages; opt-in for third-party theme/module/runner authors, with user-facing docs for them. stylelint (≥ 16), eslint (≥ 9) and `-api` are peer dependencies (so the rules match the author's catalogue). New package kind `lint`. Added by C3 (C1 unchanged). Claude proposed a private package; the user wanted third parties to be able to opt in. From C3 planning (Q4) |
| D89 | **Isolation model: the shell in Shadow DOM, showcases in the light DOM** (supersedes D44's tiers 2–3 and D87) | The module renders the shell (layout, nav, README content, tests, switcher) into a shadow root; the theme's shell stylesheet and the module's CSS are adopted there, never into `document.head`. Showcases are light-DOM children of the host (wrappers `[data-tss-showcase]`), slotted into their stage: no selector crosses either way, user CSS/CSS-in-JS and portals (modals, dropdowns, tooltips) keep working, and nothing is clipped as long as the shell's layout rules hold (document scrolling, no clipping or containing-block properties on stage ancestors, the host as a stacking context). Showcases don't inherit the user's `body` typography; the `[data-tss-showcase]` wrapper and a document-level showcase stylesheet option stand in for `body` (D90). User CSS styling `html`/`body` layout or the root font size still affects the page (documented). Inheritance is reset with `:host { all: initial !important }` (document rules beat `:host` otherwise) and `slot { all: initial !important; display: contents !important; color-scheme: <selected> !important }` (custom properties still pass); the host is the custom element `<tsshowcase-app>`. READMEs are always themed as shell content. Every showcase receives a colour scheme through a module-defined interface (React: a `colorScheme` prop) and gets its own light/dark selector defaulting to the active theme's scheme; the stage backdrop follows it. A showcase can opt, as a whole, into rendering inside the shell (needed for the D41 showcases of shell components). Shell overlays render inside the shadow root. C2 rule 7 states it mechanism-neutrally; `react-default` uses this technique. Chosen over iframes (overlays clipped) and Shadow-DOM showcases (portals escape, user CSS lands in `document.head`). The user's redefinition of the showcase styling requirements, C3 planning |
| D90 | **D89 refinements from the second C3 review** (`03b-c3-plan-review.md`) | (1) Each showcase renders into a light-DOM `[data-tss-showcase]` wrapper (named slot `tss-showcase-<n>`), playing the role of `body`; modules offer a showcase stylesheet option (document-level CSS for app typography) (H1). (2) The default theme keeps `rem` (respects the reader's font-size setting); showcases must not set the root font size (H3). (3) Dev theme-stylesheet updates are pushed through `DevContext.onThemeStylesheetsUpdate` (Vite doesn't watch `node_modules`) (H7). (4) The per-showcase scheme selector follows the theme until changed manually, resets on theme switch, isn't persisted (H12). (5) The scoped output keeps CSS nesting (H18). (6) Contrast is smoke-tested in C12 (H31, D85). Also: C2 rule 3 (binding: shadow-root shell, stylesheets adopted in order) vs rule 7 (free: how showcases are isolated); one document-level exception (`color-scheme` and canvas on `<html>`, `index.html` meta + inline script); C15 provides a framework-agnostic runtime API for stylesheets and theme state |

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

- *(D89, → C10/C15/C13)* **Isolation pitfalls of the Shadow-DOM shell:**
  - showcases don't inherit the user's `body` typography (the slot resets inheritance): use the `[data-tss-showcase]` wrapper and the showcase stylesheet option (D90);
  - document rules beat `:host` rules unless the `:host` rules are `!important`;
  - `rem` follows the document root, so a showcase setting `html { font-size }` rescales the shell; `html`/`body` layout CSS from showcases affects the page;
  - overflow clipping, containing blocks and stacking follow the flat tree: no `overflow`/`transform`/`filter`/`contain`/`container-type`/`will-change` on stage ancestors, document scrolling, host as a stacking context;
  - fragment navigation and `:target` don't reach shadow trees: the router scrolls to headings itself;
  - Vite doesn't watch `node_modules`, so theme stylesheet updates in `dev` are pushed through the contract (D90);
  - `@font-face` doesn't work inside shadow roots (fonts stylesheet in the document, D82).

- *(→ C10, D43/D48)* **Dependency resolution at build time.** Showcase pages import the user's source, so the user's `react`, `react-dom` and their versions must be used, not a copy bundled with the CLI. Two copies of React means broken hooks. The build must resolve framework packages from the showcased repo.
- *(→ C10)* **TS / path aliases in the showcased repo.** The bundler must respect the user's `tsconfig` paths, JSX settings, etc. Otherwise showcases that work in the repo fail in the showcase build.
- *(→ D5, C2 repo-level verification)* **Verification is per-directory only, but some rules are cross-cutting.** Example: "is this unit re-exported from the package entry?" needs repo-wide context. `verify` likely needs a context object (package info, entry files, the whole tree) and/or a whole-repo verification hook.
- *(resolved: D5)* **"Publicly exported" detection.** It needs a defined strategy: directory convention vs. static analysis of entry exports (TS compiler API). This affects both `check` and the hierarchy given to `build`.
- *(resolved: D40)* **Passing all file contents** for every directory can be expensive. Consider lazy content accessors.
- *(resolved: D13, D31)* **Interface extensibility and versioning.** The interface needs an `apiVersion`, optional capabilities, and input objects that can gain fields without breaking modules.
- *(resolved: D9, → C6)* **Loading custom modules and themes.** Custom modules are loaded with dynamic `import()` from a path. TS-authored modules need a loader (e.g. jiti/tsx) or must be prebuilt to JS.
- *(resolved: D46)* **Theme rule enforcement.** "No styling outside variables/classes" ideally needs tooling to enforce it (e.g. a stylelint rule banning literal values in module CSS), not just discipline.
- *(partly resolved: D8; caching → C11)* **Running tests during `build`.** It is slow and can fail. Decide whether a failing test blocks the build or is just displayed, and whether cached results can be reused.
