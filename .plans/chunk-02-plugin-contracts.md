# Chunk plan C2: Plugin contracts (`@ofzza/tsshowcase-api`)

> Status: **APPROVED** (round 5, 2026-10-18); stays open only for C3 to fill the catalogue slot (§11, D28) ( review triage in `02a-c2-plan-review.md`). Decisions referenced as `D<n>` are in AGENTS.md §3. Stays open until the C3 plan is approved (C3 fills the catalogue slot, D28).

## Goal

The complete, documented v1 contracts between the CLI and its plugins (rendering modules, test runners, themes), plus the shared data model, issue model and output inclusion rules. (The user-facing config is a CLI detail, D66, §10.) Everything lives in `@ofzza/tsshowcase-api`: TypeScript types, a handful of runtime constants and tiny helpers, no heavy dependencies.

The sketches below are the **proposal under review**. They use real TypeScript so they can be argued about precisely. JSDoc is left out of the sketches for brevity; the implementation documents every member (D10).

## Design principles

1. **Additive-only within a major (D13).** Every member added after 1.0 is optional and feature-detected. Every input and output object may gain fields. Consumers on both sides ignore fields they don't know.
2. **Plain data across the boundary.** Everything the CLI hands to `build`/`dev` (the `ShowcaseData`) is JSON-serialisable: no functions, no `Map`s, no class instances, no `Date`s (ISO strings instead). Functions only appear on _context_ objects (`read()`, `loadTestRunner()`, subscriptions), never in data.
3. **Repo-relative POSIX paths everywhere in data.** Absolute paths appear only on contexts and plugin outputs that never reach the browser (`repo.root`, `repoRoot`, `cacheDir`, `ThemeDefinition.css`). This keeps the user's filesystem layout out of the published showcase, and makes the data identical on Windows and Linux.
4. **Directories are identities.** A package is identified by its directory (`'.'` for the root), a unit by its directory (D36: one unit per directory). No separate id scheme to keep in sync.
5. **Rule violations are data, failures are exceptions.** `verify`/`analyze` return issues (D2). A module throws only for genuine failures (bugs, bad config, a crashed tool); the CLI catches, names the module, and reports.
6. **The CLI owns cross-cutting behaviour once,** so each module doesn't reimplement it: severity overrides, output-dir guarding, file watching, markdown rendering, theme resolution.
7. **The CLI performs all writes (D64).** Modules never write to the showcased repo or the output: `scaffold` and `build` _return_ files (paths + contents) and the CLI writes them. Reads should go through `RepoFiles` (D40) where practical; that is preferred, not enforced (the TS compiler in `analyze` reads the disk directly). This is a contract rule, not a sandbox: module code could still write anywhere, but the correct path is the easy one.
8. **Every method may be async** (`MaybePromise<T>`), except `resolveRoute`, which the markdown pipeline calls for every link and must stay synchronous.

## 1. Package layout (dogfooding D41)

Each exported unit is its own directory with a README (D41). Type-only units need only a README (D38); units with runtime code also get tests. Contracts are additionally locked by type tests (§12).

```
packages/tsshowcase-api/src/
  index.ts                 re-exports every unit
  version/                 IMPLEMENTED_API_VERSION                                 runtime
  common/                  MaybePromise, Logger, JsonValue, Unsubscribe            types
  schema/                  JsonSchema type + OptionsOf<TSchema> (tsschema-based, Q1) types
  data/                    the data model: repo, packages, units, docs, tests      types
  issues/                  Issue, Severity, RuleDescriptor + issue() helpers       runtime
  module/                  ShowcaseModule + its contexts                           types
  testrunner/              TestRunner + raw test result hierarchy                  types
  theme/                   ThemeSet, ThemeDefinition                               types
  catalogue/               class + CSS variable constants (slot; C3 fills it)      runtime
  inclusion/               output inclusion constants (theme attribute, storage key) runtime
```

`-api`'s only dependency is **`@ofzza/tsschema`**, used for its types (its public declarations reference them, so it must be a real dependency, not a devDependency). It does not depend on `semver` (C6's version check lives in the CLI) or on a validator: **ajv lives in the CLI** (Q1).

## 2. Versioning (D13, D31, D80)

- `IMPLEMENTED_API_VERSION` is read from `-api`'s own `package.json` (`import pkg from '../../package.json' with { type: 'json' }`; verified to work with the composite build), so there is exactly one place where the interface version lives.
- **Every package starts at `0.1.0` (D80)**, set in the C1 skeletons. The first publish (C14) is `0.1.0`, the initial release; contract changes before it are edited in place.
- **Changelog from the first release.** No changesets before the first publish; the changelog starts at "0.1.0 – initial release", and every change after it carries a changeset. **The interface changelog is `-api`'s changesets `CHANGELOG.md`** (D31: the `-api` version _is_ the interface version). One place per fact (D56). Members added in a later version say "Added in 0.x." / "Added in 1.x." in their JSDoc prose (the code style allows only `@param`/`@returns`, so there is no `@since`).
- **Pre-1.0 semver (D80).** In `0.x` a **minor** version may break the contract (field testing); patches are additive. The additive-only rule (D13) applies from `1.0.0`, a deliberate later decision. Breaking changes before 1.0 use **minor** changesets: a "major" changeset on a `0.x` package jumps straight to `1.0.0` (gotcha, documented in AGENTS.md and PUBLISH.md).
- Every plugin class declares `static readonly apiVersion: string`, a semver range (e.g. `'^0.1'`, which matches `0.1.x` only, so a module written for 0.1 is correctly rejected by a CLI implementing 0.2). The CLI checks `satisfies(IMPLEMENTED_API_VERSION, apiVersion)` at load time (C6).

## 3. Common types

```ts
export type MaybePromise<T> = T | Promise<T>;
export type Unsubscribe = () => void;
export type JsonValue = string | number | boolean | null | JsonValue[] | { [key: string]: JsonValue };

/** Repo-relative, POSIX-separated path, no leading `./`; `'.'` is the repo root. Documentation-only alias. */
export type RepoPath = string;

export interface Logger {
  debug(message: string): void;
  info(message: string): void;
  warn(message: string): void;
  error(message: string): void;
}

/** A JSON Schema (draft 2020-12): tsschema's schema type, so both a sidecar-typed JSON import and an `as const` literal are assignable. */
export type JsonSchema = TSSchemaJSONSchema; // from @ofzza/tsschema (`schema/` unit)

/** The options a module receives: inferred from its schema with tsschema, **after** ajv's `useDefaults`, so properties with a `default` are required. */
export type OptionsOf<TSchema extends JsonSchema> = TSSchemaType<TSchema> /* with defaulted properties made required */;

/** Passed to every plugin constructor. */
export interface PluginHost {
  readonly apiVersion: string; // IMPLEMENTED_API_VERSION of the running CLI
  readonly cliVersion: string;
  readonly logger: Logger; // prefixed with the plugin's name
}
```

## 4. Data model (`data/`)

The model is **multi-package from day one** (D16). A single-package repo is one package whose directory is `'.'`.

```ts
export interface ShowcaseData {
  readonly apiVersion: string; // interface version that produced this data
  readonly revision: number; // increases with every dev snapshot (0 in build); stale messages can be dropped (F32)
  readonly generatedAt: string; // ISO timestamp; honours SOURCE_DATE_EPOCH for reproducible builds
  readonly repo: RepoNode;
  readonly layout: 'single' | 'workspace';
  readonly packages: readonly PackageNode[]; // never includes the workspace root; packages analyze omitted are absent (§5.4)
  readonly meta?: { readonly [key: string]: JsonValue }; // module-private extension bag (from AnalysisResult.meta)
}

export interface RepoNode {
  readonly name: string; // root package name, or the directory name
  readonly info?: PackageInfo; // root manifest, if any
  readonly readme?: Doc; // root README.md
  readonly licence?: Doc; // root LICEN[CS]E[.md|.txt] (D12)
  readonly vcs?: VcsInfo;
  readonly route: string; // resolveRoute({ type: 'home' }), computed by the CLI (D67)
}

export interface VcsInfo {
  readonly kind: 'git';
  readonly host: 'github' | 'gitlab' | 'bitbucket' | 'other';
  readonly webUrl?: string; // e.g. https://github.com/ofzza/tsshowcase
  readonly defaultBranch?: string;
  readonly fileUrlTemplate?: string; // resolved D24 template, e.g. '{webUrl}/blob/{branch}/{path}'
}

export interface PackageNode {
  readonly dir: RepoPath; // identity; '.' in a single-package repo
  readonly info: PackageInfo;
  readonly readme?: Doc;
  readonly licence?: Doc;
  readonly children: readonly HierarchyNode[]; // top level of this package's unit tree
  readonly route: string; // computed by the CLI (D67)
  readonly meta?: { readonly [key: string]: JsonValue };
}

/** Source-agnostic package information. v1 fills it from package.json; other repo types come later. */
export interface PackageInfo {
  readonly source: 'npm' | (string & {}); // where it came from
  readonly manifest: RepoPath; // e.g. 'packages/a/package.json'
  readonly name?: string;
  readonly version?: string;
  readonly description?: string;
  readonly license?: string; // SPDX expression as declared
  readonly homepage?: string;
  readonly keywords?: readonly string[];
  readonly private?: boolean;
  readonly raw?: JsonValue; // the full manifest, for modules that need more
}

export interface Doc {
  readonly path: RepoPath;
  readonly format: 'markdown' | 'text'; // D71: `.md` files are markdown; LICENSE / .txt files are text
  readonly source: string; // the raw file contents (D23)
  readonly html: string; // CLI-rendered (C5): markdown → catalogue classes, heading ids, rewritten links; text → escaped <pre> with a catalogue class
  readonly headings: readonly DocHeading[]; // for tables of contents; empty for text docs
}

export interface DocHeading {
  readonly depth: 1 | 2 | 3 | 4 | 5 | 6;
  readonly text: string;
  readonly id: string; // the element id C5 put on the heading
}
```

### 4.1 Hierarchy and units (D36)

The tree mirrors directories. A **group** is an intermediate directory that is not itself a unit (a menu level). A **unit** may also have children (nested units, D36), so it is both a page and a parent.

```ts
export type HierarchyNode = GroupNode | UnitNode;

export interface GroupNode {
  readonly type: 'group';
  readonly dir: RepoPath;
  readonly name: string; // directory name
  readonly readme?: Doc; // D78: a README.md found in the group directory by the CLI; never required, never an issue
  readonly route?: string; // D78: set when the module gives groups pages (resolveRoute returned a URL)
  readonly children: readonly HierarchyNode[];
}

export interface UnitNode {
  readonly type: 'unit';
  readonly dir: RepoPath; // identity
  readonly name: string; // display name (module-chosen, defaults to the directory name)
  readonly kind: UnitKind; // module-chosen summary kind (D37)
  readonly exports: readonly ExportedSymbol[];
  readonly readme?: Doc; // optional in data: a missing README is an issue, not a missing page
  readonly showcases: readonly ShowcaseEntry[]; // may be empty (D37, D61)
  readonly tests: UnitTests;
  readonly children: readonly HierarchyNode[];
  readonly route: string; // computed by the CLI (D67)
  readonly meta?: { readonly [key: string]: JsonValue };
}

/** Well-known kinds; modules for other frameworks add their own (e.g. 'service', 'directive'). */
export type UnitKind = 'component' | 'hook' | 'function' | 'class' | 'constant' | 'type' | (string & {});

export interface ExportedSymbol {
  readonly name: string; // as exported from the package entry ('default' for default exports)
  readonly kind: UnitKind;
  readonly typeOnly: boolean;
  readonly source: RepoPath; // the file that declares it
  readonly entry: string; // the package export subpath it is reachable from, e.g. '.' or './hooks'
}

export interface ShowcaseEntry {
  readonly name: string; // e.g. 'Default', 'Sizes'; the module derives it
  readonly file: RepoPath; // the showcase source file; the module turns it into an import at build time
}
```

**Who builds the tree:** the module's `analyze` returns a _flat_ list of units per package plus that package's `unitRoot` (e.g. `src`). The CLI builds the tree from the directories between `unitRoot` and each unit (groups for non-unit levels, nesting for units inside units). This keeps the tree consistent across modules, and keeps `src/` out of the menu.

### 4.2 Test results in the data

```ts
export type UnitTests =
  | { readonly status: 'not-run' } // `--no-tests`, or the module has no runTests
  | { readonly status: 'unavailable'; readonly reason: string } // no runner detected, or the runner crashed (D42)
  | { readonly status: 'available'; readonly summary: TestSummary; readonly files: readonly TestFileResult[] };

export interface TestSummary {
  readonly total: number;
  readonly passed: number;
  readonly failed: number;
  readonly skipped: number; // includes todo
  readonly running: number; // > 0 only in dev while a watch run is in progress
  readonly durationMs?: number;
}
```

`TestFileResult` is the runner's raw shape (§6). A unit with no test files is `available` with zero files; whether that is acceptable is a `verify` concern (D37/D38), not a data concern. A unit **absent** from the module's `TestResults.units` gets `available` with zero files when the runner ran, otherwise the overall status (`not-run` / `unavailable`).

### 4.3 Root vs package docs, and the single-package collapse

- `repo.readme` / `repo.licence` are the root files. `package.readme` / `package.licence` are each package's own files.
- **Single-package repo** (`layout: 'single'`): `packages[0].dir === '.'`, and its `readme`/`licence` are the _same_ docs as the repo's. Modules must collapse the package level (D19): home shows the README once, units hang directly under home.
- **Workspace** (`layout: 'workspace'`): the workspace root is never a package (even if it isn't `private`). The home page shows `repo.readme`.
- **No manifest, no package:** a repo without a root manifest (and no workspaces) has `packages: []`; `repo.name` is the directory name. The module may still analyze, but there is nothing to showcase.
- **No inheritance in the data:** a workspace package without a licence has `licence: undefined`. A module may choose to fall back to `repo.licence` and should say so on the page.

## 5. Rendering module interface (`module/`)

### 5.1 The class

The module's default export is a class (D9). TypeScript can't check static members through `implements`, so `-api` exports the constructor type and authors check against it with `satisfies` (documented in the README sample).

```ts
export interface ShowcaseModuleConstructor<TOptions = unknown> {
  readonly apiVersion: string; // semver range (D13)
  readonly displayName?: string; // for logs and reports; defaults to the reference it was loaded by
  readonly optionsSchema?: JsonSchema; // JSON Schema (draft 2020-12) for moduleOptions: a JSON import or an `as const` literal (D6, Q1)
  readonly rules?: readonly RuleDescriptor[]; // rule ids it can emit, for config validation and docs
  readonly optionFlags?: { readonly [flag: string]: string }; // D79: CLI flag name → dot path into the options (§5.11)
  new (init: ModuleInit<TOptions>): ShowcaseModule;
}

export interface ModuleInit<TOptions> extends PluginHost {
  readonly options: TOptions; // the root moduleOptions, validated against optionsSchema, defaults applied; per-package values: PackageInput.options (D69)
  readonly repoRoot: string; // absolute
}
```

The CLI constructs **one instance per CLI invocation** and calls its methods in the order of §5.9. Methods may be called concurrently (e.g. `verify` for many directories at once), so a module must not keep per-call state on `this`.

### 5.2 Methods: which are required

```ts
export interface ShowcaseModule {
  // Required
  analyze(ctx: AnalyzeContext): MaybePromise<AnalysisResult>;

  // Optional, feature-detected (the CLI reports a clear "module X does not support Y")
  verify?(dir: DirectoryInput, ctx: VerifyContext): MaybePromise<VerifyResult>; // required for check
  resolveRoute?(target: RouteTarget, ctx: RouteContext, heading?: string): string | undefined; // required for build/dev
  runTests?(ctx: TestContext): MaybePromise<TestResults>;
  build?(ctx: BuildContext): MaybePromise<BuildResult>;
  dev?(ctx: DevContext): MaybePromise<DevSession>;
  scaffold?(ctx: ScaffoldContext): MaybePromise<ScaffoldResult>;
  dispose?(): MaybePromise<void>;
}
```

Rationale (D63): **only `analyze` is required**, because every action starts with it. Every other method is optional, and an action whose method is missing fails with a clear "module X does not support `<action>`" error. A stub (a `verify` that always passes, a fake `build`) would be worse than that error: a stub `verify` silently passes CI. Partial modules are legitimate (a check-only module for a framework with no renderer yet, a build-only module for repos that don't follow the conventions), and so is a module that supports nothing beyond `analyze`; it is simply useless, and every action says so. Action prerequisites are checked when the action starts: `check` needs `verify`; `build`/`dev` need `resolveRoute` plus `build`/`dev`; `init` needs `scaffold`.

### 5.3 Shared context pieces

```ts
/** Lazy, cached, read-only view of the repo (D40). Paths are repo-relative; a path resolving outside the repo root is rejected. The CLI records reads for dev invalidation. */
export interface RepoFiles {
  read(path: RepoPath): Promise<string>; // utf-8; rejects if missing
  exists(path: RepoPath): Promise<boolean>;
  list(dir: RepoPath): Promise<DirectoryListing>; // non-ignored entries only
}

// `list` is the gitignore-aware view of the repo for everything outside `verify` (runner `detect`, convention-based `analyze`, `scaffold`);
// plain `fs` would descend into ignored directories and bypass dev read tracking.
// No `readJson`: manifests are already in `PackageInfo.raw`, tsconfig files are JSONC, and `JSON.parse(await read(p))` is trivial.
// It can be added later as an optional member (D13); it could never be removed.

export interface DirectoryListing {
  readonly files: readonly string[]; // names, not paths
  readonly directories: readonly string[];
}

/** The repo as scanned by the CLI (C4), before the module has said anything about units. */
export interface RepoInput {
  readonly root: string; // absolute
  readonly name: string;
  readonly info?: PackageInfo;
  readonly vcs?: VcsInfo;
  readonly layout: 'single' | 'workspace';
  readonly packages: readonly PackageInput[];
  readonly files: RepoFiles;
}

export interface PackageInput {
  readonly dir: RepoPath;
  readonly info: PackageInfo;
  readonly readme?: RepoPath;
  readonly licence?: RepoPath;
  readonly options: unknown; // D69: root moduleOptions merged with this package's own, validated against optionsSchema (typed by the module as its TOptions)
}
```

### 5.4 `analyze` (D5) and repo-level verification

Runs once per `check`/`build`/`dev`, and again in `dev` when the structure changes or any of its `inputs` changes. Typically uses the TS compiler API against the real filesystem (`repo.root`), so its reads are not tracked through `RepoFiles`; instead it **reports the files it depended on** (`inputs`, D68), e.g. `program.getSourceFiles()` filtered to the repo. That catches content edits such as a new `export *` in an entry file, which turn a directory into a unit without any structural change.

```ts
export interface AnalyzeContext {
  readonly repo: RepoInput;
  readonly signal: AbortSignal; // dev aborts a stale analysis
}

export interface AnalysisResult {
  readonly packages: readonly PackageAnalysis[]; // one per analysed package; an omitted package is hidden from the showcase (D72)
  readonly issues?: readonly Issue[]; // repo-/package-level issues (e.g. unresolvable entry, export whose source is shared)
  readonly inputs?: readonly RepoPath[]; // files the analysis depended on; dev re-analyses when any changes (D68)
  readonly state?: unknown; // opaque, handed back to verify/resolveRoute/runTests/scaffold; never serialised
  readonly meta?: { readonly [key: string]: JsonValue }; // becomes ShowcaseData.meta
}

export interface PackageAnalysis {
  readonly dir: RepoPath;
  readonly unitRoot?: RepoPath; // directory the menu tree starts at; defaults to the package dir
  readonly units: readonly AnalyzedUnit[];
  readonly meta?: { readonly [key: string]: JsonValue };
}

export interface AnalyzedUnit {
  readonly dir: RepoPath;
  readonly name?: string;
  readonly kind: UnitKind;
  readonly exports: readonly ExportedSymbol[];
  readonly readme?: RepoPath; // where the module expects the README; the CLI reads and renders it if it exists (missing = no UnitNode.readme, not a CLI error; verify reports it)
  readonly showcases?: readonly ShowcaseEntry[];
  readonly meta?: { readonly [key: string]: JsonValue };
}
```

**Repo-level verification is covered by `analyze` returning issues**, rather than a separate hook. Each issue may carry a `dir` (attached to that directory in the report) or none (shown at repo/package level, using `package` when set). One pass, no duplicated analysis.

**Omitting a package (D72).** A module that wants a package visible without units returns it with `units: []` (its README and licence still get a page). A package left out of `packages` entirely is **hidden** from the showcase: it is absent from `ShowcaseData.packages`. `check` still visits its directories (the report covers every non-ignored directory); whether they pass is up to `verify`.

**The CLI validates the analysis (D70).** A unit directory that doesn't exist, lies outside its package or its `unitRoot`, or appears twice, and a `PackageAnalysis.dir` that isn't a known package, are module bugs: the CLI throws an error naming the module.

### 5.5 `verify` (D2, D40)

Called for **every non-ignored directory** of the repo (requirement), after `analyze`.

```ts
export interface DirectoryInput {
  readonly path: RepoPath;
  readonly package?: RepoPath; // the nearest package directory that owns it, if any
  readonly files: readonly string[]; // own listing, eager (every verify needs it): names in this directory, non-ignored
  readonly directories: readonly string[]; // names of non-ignored subdirectories
  read(path: string): Promise<string>; // lazy and cached (D40)
  exists(path: string): Promise<boolean>;
  list(path?: string): Promise<DirectoryListing>; // omitted = this directory (same as files/directories)
  resolve(path: string): RepoPath; // a path relative to this directory → repo-relative (for Issue.file)
}

export interface VerifyContext {
  readonly repo: RepoInput;
  readonly analysis: AnalysisResult;
  readonly unit?: AnalyzedUnit; // the unit at this directory, if analyze found one
}

export type VerifyResult = { readonly ignored: true } | { readonly ignored?: false; readonly issues: readonly Issue[] };
```

A directory that passes returns `{ issues: [] }` (`ignored` is optional, so `{ ignored: false, issues: [] }` means the same). Issues that are all warnings still pass: `check` fails only on `error`s left after the severity overrides (§9).

`DirectoryInput` is a **scoped view of the same `RepoFiles`** (same cache, same read tracking), plus the directory's own eager listing. Its paths resolve **relative to this directory**: `'README.md'`, child paths (`'__tests__/a.test.ts'`) and parent paths (`'../index.ts'`) are all allowed. As with `RepoFiles`, the only boundary is the repo root: a path resolving outside it is rejected. Restricting `..` would protect nothing (`ctx.repo.files` reaches the whole repo anyway), and parent-looking rules exist (e.g. nested units, D36). Reads through either object are tracked, so `dev` invalidates a directory's result when any file it read changes.

The repo root (`'.'`) is verified too. An ignored directory still shows in the report tree (greyed out, "ignored by module"), since the requirement asks for the full non-ignored structure. Only the scan removes a directory from the tree entirely: `.gitignore` (incl. nested files and `.git/info/exclude`), the always-ignored `.git`, `node_modules` and output directory (C4), config `exclude` patterns, and skipped packages (D69).

**A throwing `verify`** doesn't abort `check`: the CLI turns it into a synthetic `module/error` issue (severity `error`, the message names the module) on that directory, continues with the others, and `check` exits non-zero.

### 5.6 `resolveRoute` (D35)

```ts
export type RouteTarget =
  | { readonly type: 'home' }
  | { readonly type: 'package'; readonly package: RepoPath }
  | { readonly type: 'group'; readonly package: RepoPath; readonly dir: RepoPath } // D78
  | { readonly type: 'unit'; readonly package: RepoPath; readonly unit: RepoPath };

export interface RouteContext {
  readonly analysis: AnalysisResult; // the analysis the route belongs to (in dev, the current one)
}

// resolveRoute(target, ctx, heading?) → an in-app URL relative to the app's index.html (e.g. '#/button', '#/button?h=props'), or undefined (D78)
```

- **Signature (D67):** `resolveRoute(target: RouteTarget, ctx: RouteContext, heading?: string): string`. Like every other method, it gets the analysis through its context, so a module never keeps the last analysis on `this` (unsafe when `dev` re-analyses while calls are in flight). This revises D35's `resolveRoute(target, heading?)`.
- `heading` is a heading id from `Doc.headings` (C5 generates the ids, so the module and the markdown pipeline agree).
- In a unit target, `package` is a convenience: it must be the unit's owning package.
- The returned URL must work under any base path (D43 inclusion rules): relative, no leading `/`.
- **`undefined` means "no in-app page" (D78).** A module that doesn't give groups pages returns `undefined` for group targets; the CLI then leaves `GroupNode.route` unset and links to that group's README go to the VCS host. Home, package and unit targets must resolve to a URL (`undefined` there is a module error).
- **Purity:** the result depends only on its arguments.
- **Routes live in the data (D67).** The CLI calls `resolveRoute` for home, every package and every unit and stores the result as `route` on `RepoNode` / `PackageNode` / `UnitNode`; it also calls it for README links and heading anchors (C5). The browser app reads routes from the data instead of re-deriving them: one source of truth.
- **Link mapping for C5 (D73):** a link to a unit's README **or its directory** → that unit; a package's README or directory → that package; a group's README or directory → that group if it has a route, else the VCS host (D78); the root README or the repo root → home; anything else in the repo (including a unit's `index.ts`) → the VCS host URL (D24); external links are untouched.

### 5.7 `runTests` (D3, D34, D42, D18)

```ts
export interface TestContext {
  readonly repo: RepoInput;
  readonly analysis: AnalysisResult;
  readonly mode: 'run' | 'watch';
  readonly signal: AbortSignal; // in watch mode, aborting stops watching
  loadTestRunner(ref?: string, options?: LoadTestRunnerOptions): Promise<LoadedTestRunner | undefined>; // D74
  onAnalysisUpdate(listener: (analysis: AnalysisResult) => void): Unsubscribe; // watch mode: called after every dev re-analysis (D75)
  reportTestUpdate(update: TestResultsUpdate): void; // watch mode only: push mapped updates to the CLI
}

export interface LoadTestRunnerOptions {
  readonly candidates?: readonly string[]; // restricts auto-detection to these runner names/references (D42, D74)
}

export interface LoadedTestRunner {
  readonly name: string; // as resolved, e.g. 'vitest'
  readonly runner: TestRunner; // already constructed and version-checked
}

export interface TestResults {
  readonly units: { readonly [unitDir: RepoPath]: UnitTests };
  readonly unmapped?: readonly TestFileResult[]; // test files outside any unit, for diagnostics
}

export interface TestResultsUpdate {
  readonly revision: number; // increasing per dev session; the CLI drops out-of-order updates (F32)
  readonly units: { readonly [unitDir: RepoPath]: UnitTests }; // only the units that changed; full replacement per unit
}
```

**`loadTestRunner` (D74):** with a `ref`, the CLI resolves it (D9), instantiates and version-checks it. Without one, it auto-detects: it calls `detect` on every candidate (the built-in runners plus the config's `testRunners`, intersected with `candidates` when given). Exactly one match is used and logged; **several matches throw** (a configuration error the user must fix, with a hint to set `testRunner`); **no match resolves `undefined`** (a legitimate state: the module marks its units `unavailable`). A `ref` that can't be resolved or fails the version check also throws.

**Re-analysis in watch mode (D75):** `onAnalysisUpdate` hands the module every new `AnalysisResult` after a `dev` re-analysis. The module rebuilds its file → unit mapping and re-emits the remapped results through `reportTestUpdate`; the runner keeps running (no Vitest restart):

```ts
async runTests(ctx: TestContext) {
  let units = mapUnits(ctx.analysis);
  ctx.onAnalysisUpdate((analysis) => {
    units = mapUnits(analysis);
    ctx.reportTestUpdate({ revision: next(), units: remapAll(units) });
  });
  const loaded = await ctx.loadTestRunner(this.options.testRunner, { candidates: ['vitest'] });
  // … loaded.runner.watch(…, (update) => ctx.reportTestUpdate(mapToUnits(update, units)))
}
```

In `watch` mode `runTests` resolves with the first complete result and keeps calling `reportTestUpdate` until `signal` aborts. (Renamed from D42's `ctx.onTestUpdate`: `on…` names are subscriptions elsewhere, and `DevContext.onTestUpdate` is the opposite direction, CLI → dev server.)

Paths: `TestFileResult.file` is always repo-relative, even when `TestRunOptions.root` is a package directory.

### 5.8 `build` and `dev` (D43)

```ts
export interface BuildContext {
  readonly data: ShowcaseData;
  readonly themes: readonly ResolvedTheme[]; // in config order
  readonly defaultTheme?: string; // explicit config choice; otherwise the inclusion rules decide (§8)
  readonly repoRoot: string; // absolute; to turn RepoPaths (showcase files) into imports
  readonly cacheDir: string; // absolute, module-private scratch space, persists across runs
  readonly signal: AbortSignal;
}

export interface BuildResult {
  readonly files: readonly BuildFile[]; // the whole site; the CLI writes it (D64)
}

export interface BuildFile {
  readonly path: string; // relative to the site root, POSIX; no '..', no absolute paths (the CLI rejects them)
  readonly contents: string | Uint8Array;
}

export interface DevContext extends BuildContext {
  readonly host?: string;
  readonly port?: number;
  onDataUpdate(listener: (data: ShowcaseData) => void): Unsubscribe; // full snapshot each time (Q6)
  onTestUpdate(listener: (update: TestResultsUpdate) => void): Unsubscribe;
}

export interface DevSession {
  readonly url: string;
  readonly closed: Promise<void>; // settles when the server stops; rejects if it fails after starting
  close(): Promise<void>;
}
```

- **The module returns the site; the CLI writes it (D64).** `build` doesn't know where the site will end up (URLs are relative anyway, §8 rule 1), so `BuildContext` has no `outDir`. The CLI validates the returned paths, checks the inclusion rules it can check (e.g. `index.html` exists, no absolute build-machine paths, §8 rules 1–2), and writes `outDir` with the output guard, an atomic swap (temp dir, then replace, so a failed build never leaves a half-written site) and its marker file (C11). Later minors may add CLI post-processing of the files (lint, format, minify, `.nojekyll`) before writing.
- **Toolchains:** Vite builds in memory (`build.write: false` returns chunks and assets), which `-base-vite` maps to `BuildFile[]` (C15). A toolchain that can only write to disk builds into `ctx.cacheDir` and returns what it wrote. A showcase site is a few MB, occasionally tens, so holding it in memory is fine.
- `dev` writes nothing: dev servers serve from memory.
- `data` already contains test results (or `not-run`/`unavailable`), README HTML and the tree.
- In `dev`, test updates reach the dev server separately (`onTestUpdate`) so a test run doesn't force a full data push. The CLI also merges them into its own copy, so the next full `onDataUpdate` snapshot is consistent.
- **Shutdown order** (Ctrl-C or a fatal error): the CLI aborts every `signal` (stopping analysis and test watching) → `DevSession.close()` → `dispose()`. A rejected `DevSession.closed` ends `dev` with the error.
- Theme CSS edits in `dev` are the module's dev server's job (it watches the `css` files it was given). Changing which themes are configured needs a restart.

### 5.9 Call order per action

| Action  | Calls                                                                                                                                                                 |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `check` | `analyze` → `verify` × every directory → severity overrides applied by the CLI → report                                                                               |
| `init`  | `analyze` → `scaffold` → the CLI writes the returned files                                                                                                            |
| `build` | **the full `check`** (D76: `analyze` + `verify` × every directory, report printed; errors fail the build unless `--no-check`) → `runTests('run')` (unless `--no-tests`) → CLI reads READMEs, renders HTML and computes routes via `resolveRoute` → `build` |
| `dev`   | as `build` (check issues shown, never fatal, D76), but `runTests('watch')` and `dev`; on file changes the CLI re-renders / re-analyses and pushes `onDataUpdate`; test updates flow `reportTestUpdate` → CLI → `onTestUpdate` (D42) |
| all     | `dispose` at the end, also on Ctrl-C                                                                                                                                  |

### 5.10 `scaffold` (D16)

The module plans the files; the **CLI writes them** (one implementation of dry-run, never-overwrite-silently and reporting). C8 decides `init` semantics (new unit vs filling gaps, adding the entry export).

```ts
export interface ScaffoldContext {
  readonly repo: RepoInput;
  readonly analysis: AnalysisResult;
  readonly target: RepoPath; // the directory passed to `tsshowcase init <dir>`
  // no `args`: module-defined flags (e.g. --kind hook) arrive as options via optionFlags (D79)
}

export interface ScaffoldResult {
  readonly files: readonly ScaffoldFile[];
  readonly notes?: readonly string[]; // printed after writing ("remember to …")
}

export interface ScaffoldFile {
  readonly path: RepoPath;
  readonly contents: string;
  readonly ifExists: 'skip' | 'overwrite'; // 'overwrite' is for edits the module computed from the current contents (e.g. the entry file)
}
```

### 5.11 Module options (D6)

Validated by the CLI with ajv against `optionsSchema` (Q1), with schema `default`s applied (`useDefaults`). Without a schema the options are passed through as given. Authors derive their options type from the schema rather than writing it twice. **Our own modules keep the schema as a plain `options.schema.json` with a tsschema sidecar** (D62, the same convention as the CLI's config schemas); third-party authors may do the same or write an `as const` literal; the contract only needs the value.

```ts
import optionsSchema from './options.schema.json' with { type: 'json' }; // typed by the committed options.schema.d.json.ts sidecar
type Options = OptionsOf<typeof optionsSchema>; // via tsschema's TSSchemaType: e.g. { testRunner?: string }
export default class MyModule implements ShowcaseModule {
  static readonly apiVersion = '^0.1';
  static readonly optionsSchema = optionsSchema;
  constructor(init: ModuleInit<Options>) {}
  // …
}
MyModule satisfies ShowcaseModuleConstructor<Options>; // checks the static side
```

**Module-defined CLI flags (D79).** A module maps its own CLI flags onto option paths:

```ts
static readonly optionFlags = { 'test-runner': 'testRunner', kind: 'scaffold.kind' } as const;
```

- **Precedence:** root config < package config < CLI flags (D6). The CLI applies flags on top of the root options and every merged per-package result, then validates, so every method simply sees options (`ModuleInit.options`, `PackageInput.options`).
- **Parsing from the schema** at the target path: booleans become `--flag` / `--no-flag`, numbers are coerced, arrays come from repeating the flag, enums are validated by ajv.
- **`--help`** lists the module's flags with the target property's schema `description`.
- **Collisions:** a flag that clashes with a CLI flag is a load-time error naming the module; the CLI reserves its own names.
- Flags are accepted on every action, even where unused. Options that only make sense as flags (e.g. `scaffold.kind`) are still valid in config files; their schema `description` says so.

`testRunner` (D34) is a **module option by convention**: modules that run tests should accept `testRunner?: string` and pass it to `ctx.loadTestRunner()`.

## 6. Test-runner interface (`testrunner/`)

```ts
export interface TestRunnerConstructor {
  readonly apiVersion: string;
  readonly displayName?: string;
  new (host: PluginHost): TestRunner;
}

export interface TestRunner {
  detect(repo: RunnerRepoInput): MaybePromise<boolean>; // D42 auto-detection
  run(options: TestRunOptions): Promise<RawTestReport>;
  watch(options: TestRunOptions, onUpdate: (update: RawTestUpdate) => void): Promise<RawTestReport>; // resolves with the first full run; stops on options.signal
}

export interface RunnerRepoInput {
  readonly root: string; // absolute
  readonly packages: readonly PackageInput[];
  readonly files: RepoFiles;
}

export interface TestRunOptions {
  readonly root: string; // absolute; where to run the showcased repo's own tool
  readonly files?: readonly RepoPath[]; // optional filter
  readonly signal: AbortSignal;
}

export interface RawTestReport {
  readonly status: 'completed' | 'error'; // 'error' = the runner itself failed (D42: units become 'unavailable')
  readonly error?: TestError;
  readonly files: readonly TestFileResult[];
}

export interface RawTestUpdate {
  readonly files: readonly TestFileResult[]; // full replacement per listed file
  readonly removed?: readonly RepoPath[];
}

export interface TestFileResult {
  readonly file: RepoPath;
  readonly status: TestStatus;
  readonly durationMs?: number;
  readonly error?: TestError; // file-level failure (e.g. a syntax error); its tests are then absent
  readonly children: readonly TestNode[];
}

export type TestNode =
  | { readonly type: 'suite'; readonly name: string; readonly status: TestStatus; readonly children: readonly TestNode[] }
  | { readonly type: 'test'; readonly name: string; readonly status: TestStatus; readonly durationMs?: number; readonly error?: TestError };

export type TestStatus = 'passed' | 'failed' | 'skipped' | 'todo' | 'running' | 'queued';

export interface TestError {
  readonly message: string;
  readonly stack?: string;
  readonly expected?: string;
  readonly actual?: string;
}
```

- **Never throws on failing tests (D8).** A failing test is `status: 'failed'`. A runner that can't run at all returns `status: 'error'` rather than throwing, where it can.
- Type tests (`*.test-d.ts`, D38) appear as ordinary files and tests.
- The runner knows nothing about units; the module maps (D42).

## 7. Theme contract (`theme/`)

A theme reference (D9) resolves to a module whose default export is a **theme set**: one class can provide several themes (`-theme-default` provides `default-light` and `default-dark`).

```ts
export interface ThemeSetConstructor {
  readonly apiVersion: string;
  readonly displayName?: string;
  new (host: PluginHost): ThemeSet;
}

export interface ThemeSet {
  readonly themes: readonly ThemeDefinition[];
}

export interface ThemeDefinition {
  readonly name: string; // unique within a build; the value of the theme attribute (§8)
  readonly label?: string; // shown in the switcher; defaults to name
  readonly scheme: 'light' | 'dark'; // drives the prefers-color-scheme default and `color-scheme` (D7, D44)
  readonly css: readonly string[]; // absolute paths to the theme's CSS files, in cascade order; unscoped (see below)
}

export interface ResolvedTheme extends ThemeDefinition {
  readonly reference: string; // the config reference it came from
}
```

**Reference syntax:** `<ref>` selects every theme in the set; `<ref>#<name>` selects one. Built-in names (`default-light`, `default-dark`) are aliases the CLI maps to `@ofzza/tsshowcase-theme-default#default-light` etc.

**Names and assets.** Two configured themes with the same `name` are a CLI error. `url()` references in theme CSS (fonts, images) resolve relative to their CSS file, and the build must bundle them into the output (§8 rule 2 bans third-party origins).

**Theme CSS is unscoped; the build scopes it (D65).** A theme's CSS files must not reference the theme's own name or `THEME_ATTRIBUTE`. The build scopes each theme's files under its `name` (§8 rule 3). C3 decides the mechanics (how the scoping is applied, and which selector the unscoped files are written against) and everything about the variable hierarchy.

**Extending a theme is composition, not a contract feature (D65).** There is no `extends` field. A theme package that builds on another depends on it as an ordinary npm dependency (so npm resolution, pinned by its own range, picks the actual package) and puts the base theme's CSS files before its own overrides:

```ts
// my-ocean-theme/src/index.ts  (package.json: "dependencies": { "@ofzza/tsshowcase-theme-default": "^0.1" })
import { fileURLToPath } from 'node:url';
import DefaultThemes from '@ofzza/tsshowcase-theme-default';
import type { PluginHost, ThemeSet, ThemeDefinition } from '@ofzza/tsshowcase-api';

export default class OceanThemes implements ThemeSet {
  static readonly apiVersion = '^0.1';
  readonly themes: readonly ThemeDefinition[];

  constructor(host: PluginHost) {
    const base = new DefaultThemes(host).themes.find((theme) => theme.name === 'default-light')!;
    this.themes = [{ name: 'ocean', label: 'Ocean', scheme: 'light', css: [...base.css, fileURLToPath(new URL('./ocean.css', import.meta.url))] }];
  }
}
```

`ocean.css` overrides only what it needs (e.g. the primary/secondary base tokens), and the variable hierarchy cascades it (C3). Config: `themes: ['my-ocean-theme']`, or mixed with built-ins. This only works because of the scoping rule above: self-scoped base files would keep targeting `default-light`. The `theme/` README documents the pattern (the sample is type-checked, D56). A CLI-resolved `extends` reference could still be added later, additively (D13), if composition proves clumsy.

## 8. Output inclusion rules (`inclusion/`, D43)

Binding on **every** module's output, whether or not it uses `-base-vite`. Documented in the `inclusion/` README; constants exported so nobody hardcodes the strings.

**Not a rule: how the data is embedded (D77).** How `ShowcaseData` gets into the site (compiled into the bundle, a fetched `data.json`, inlined in `index.html`, …) is the module's business: nothing outside the module's own app reads it. Rules 1 and 2 (static and relocatable, no leaked absolute paths) are the only constraints. `-base-vite` compiles it into the bundle as a virtual module (C15).

0. **The CLI alone writes the output (D64).** `build` returns the site as `BuildFile[]`; a module that writes the output directory itself breaks the contract.
1. **Static and relocatable.** The returned files form a self-contained static site with `index.html` at its root. It works when served from any sub-path (GitHub Pages project sites): all URLs relative, routing needs no server rewrites (hash routing or equivalent).
2. **No leaks.** No absolute filesystem paths from the build machine appear in the output. No runtime requests to third-party origins (fonts, CDNs, analytics) unless the user's own showcase code makes them.
3. **Every configured theme is included,** each theme's (unscoped, §7) CSS scoped by the build under its `name`, and a theme is applied by setting `THEME_ATTRIBUTE` (`'data-theme'`) to its `name` on the `<html>` element (C3 confirms the element and the scoping mechanics).
4. **Initial theme selection,** applied before first paint (no flash): the stored choice (`localStorage[THEME_STORAGE_KEY]`, `'tsshowcase:theme'`) if it names an included theme → else the configured `defaultTheme` → else the first theme whose `scheme` matches `prefers-color-scheme` → else the first theme. With only one theme, no switcher is rendered.
5. **The switcher** stores the user's choice under `THEME_STORAGE_KEY` and uses the catalogue's switcher class (C3).
6. **README HTML is inserted verbatim** inside an element carrying the catalogue's content region class (`tss-content`, D44). It is therefore fully themed: the region styles its bare elements (headings, paragraphs, links, tables, code, …) with the same classes' and variables' values the shell uses, and links and heading ids stay intact.
7. **Showcases get the theme, but only on request (D44's three tiers).** Inside a showcase:
   - **theme variables** (`--tss-…`) are inherited, so `var(--tss-…)` works and follows the active theme;
   - **catalogue classes** (`tss-title`, `tss-link`, …) work on any element and look exactly as they do in the shell;
   - **a region class** (`tss-content`) may be used to opt a subtree into themed bare elements, exactly like README content;
   - **nothing else applies:** shell CSS never targets bare elements outside a region (no `p { … }`, no `.tss-nav a` reaching in), so a user's component that uses none of the above looks exactly as it does in the user's own app.
   The theme's own rules enforce the last point (C3 stylelint, D44/D46); the module must not add CSS that breaks it. Anything beyond that (e.g. extra framework-level isolation) is the module's business.

## 9. Issue model (`issues/`)

```ts
export type Severity = 'error' | 'warning';

export interface Issue {
  readonly rule: string; // namespaced id, e.g. 'react/readme-required', 'headless/render-pure'
  readonly severity: Severity; // the module's default; the CLI applies config overrides (below)
  readonly message: string; // one sentence, no trailing period
  readonly dir?: RepoPath; // set for analyze issues that belong to a directory; verify issues get it from context
  readonly package?: RepoPath; // derived by the CLI from dir/file when unset (D70)
  readonly file?: RepoPath; // always repo-relative; in verify use `dir.resolve('README.md')`
  readonly line?: number;
  readonly hint?: string; // how to fix, e.g. 'run `tsshowcase init src/Button`'
}

export interface RuleDescriptor {
  readonly id: string;
  readonly description: string;
  readonly defaultSeverity: Severity;
  readonly preset?: string; // e.g. 'headless' (D47)
}

// runtime helpers
export const issue: {
  error(rule: string, message: string, extra?: Partial<Omit<Issue, 'rule' | 'severity' | 'message'>>): Issue;
  warning(rule: string, message: string, extra?: Partial<Omit<Issue, 'rule' | 'severity' | 'message'>>): Issue;
};
```

- **Severity overrides are applied by the CLI, not the module.** Config `rules: { 'react/showcase-required': 'off' | 'warning' | 'error' }`. `'off'` drops the issue. One implementation for every module; modules just emit their defaults. Unknown rule ids in config produce a warning when the module declares `rules`.
- **Per-package rules (D59, D70):** an issue's package (its `package`, else derived from `dir`/`file`) selects which package config's `rules` apply on top of the root ones.
- `check` exits non-zero when any `error` remains after overrides (D16).
- **`check` is a prerequisite of `build` and `dev` (D76).** Both run the full check first and print the report. In `build`, `error`s fail the build by default (`--no-check`, config `check: false`, turns that off, e.g. while adopting tsshowcase); warnings never fail. In `dev`, issues are shown but never fatal. A module without `verify` (D63) skips the verify pass with a one-line note; `analyze` issues are still printed. Tests stay as D8 says: never blocking unless `--fail-on-test-failure`.

## 10. Config: a CLI detail (D66), handed over to C6

**Not part of `-api`.** The config is user-facing and consumed only by the CLI: no plugin ever sees it (a module receives only its validated options, `ModuleInit.options`; runner candidates and severity overrides are handled inside the CLI). So, like the check report (D60), it lives in the CLI package `@ofzza/tsshowcase` and is documented in the CLI README: users write `import { defineConfig } from '@ofzza/tsshowcase'` and point `$schema` at `./node_modules/@ofzza/tsshowcase/config.schema.json`. Plain `tsshowcase.config.json` stays fully supported (D6); `defineConfig` is optional typing sugar for `.ts`/`.js` configs (the Vite/Vitest convention; the CLI validates every format the same way). C6 implements it; the design agreed so far is kept here as C6's starting point:

**The JSON Schema is the source of truth (D58).** The schemas are **plain JSON files**, `config/config.schema.json` and `config/package-config.schema.json`, each with a committed tsschema sidecar (`*.d.json.ts`, `tsschema generate --mode sidecar`) that gives the JSON import its literal types. The CLI package exports them as values (`configSchema`, `packageConfigSchema`), and the TS types below are **inferred** from them with tsschema, never written by hand. Documentation lives in each property's `description` (JSON has no comments), which editors show on hover. The shapes the inferred types must have:

```ts
export interface TsshowcaseConfig {
  module?: string; // reference (D9); default 'react-default'
  moduleOptions?: { [key: string]: unknown }; // validated by the module's optionsSchema
  themes?: string | readonly string[]; // references, optionally '#name' (§7); default ['default-light', 'default-dark']
  defaultTheme?: string; // a theme name
  outDir?: string; // build output, relative to the config file; default './showcase'
  exclude?: readonly string[]; // gitignore-style patterns, on top of .gitignore (e.g. 'fixtures/')
  rules?: { [ruleId: string]: Severity | 'off' };
  testRunners?: readonly string[]; // extra auto-detection candidates (D42)
  tests?: boolean; // false = --no-tests
  failOnTestFailure?: boolean; // D8 strict mode
  vcs?: { host?: VcsInfo['host']; webUrl?: string; defaultBranch?: string; fileUrlTemplate?: string }; // D24 overrides
}

/** A workspace package's own tsshowcase.config.* (Q3): only package-scoped keys; repo-wide keys there are an error. */
export interface PackageConfig {
  skip?: boolean; // D69: leave the package out entirely (dropped from RepoInput.packages, its directories from the scan); renamed from `exclude` (F27)
  rules?: { [ruleId: string]: Severity | 'off' }; // merged over the root rules
  moduleOptions?: { [key: string]: unknown }; // shallow-merged over the root moduleOptions
}

export function defineConfig(config: TsshowcaseConfig): TsshowcaseConfig; // identity, for typing in tsshowcase.config.ts
```

- CLI flags override file values (D6). C6 validates config files with ajv against these schemas; normalisation (e.g. `themes: string` → `[string]`) and checks JSON Schema can't express (valid semver ranges, resolvable references) happen in code after validation.
- The JSON files are **published as they are**, as extra `exports` subpaths of the CLI package (`@ofzza/tsshowcase/config.schema.json`, `…/package-config.schema.json`), so `tsshowcase.config.json` gets editor completion via `$schema`, and the CLI README links them. No emit step.
- Sidecars must be copied into `dist`, or consumers' inferred types silently become `never`: see C1 amendment A1 (D62).
- **Known gap (temporary):** until tsschema infers records (`additionalProperties`) and tuples, map-shaped fields (`rules`, `moduleOptions`) infer as `{ [key: string]: unknown }`. We accept that: the schemas (and so runtime validation) are already exact, and tightening an inferred _input_ type later only rejects configs that ajv already rejects at runtime. Code that reads those maps (C6, C8) narrows at the point of use. Must be resolved before `1.0.0` (not required for the `0.x` releases, D80).

## 11. Theme catalogue slot (`catalogue/`, D28)

C2 provides the **mechanism**; C3 fills in the names and documents the hierarchy.

```ts
export const tssClass = {
  content: 'tss-content', // region class (D44), needed by inclusion rule 6
  themeSwitcher: 'tss-theme-switcher', // needed by inclusion rule 5
  // … C3 adds the rest
} as const;

export const tssVar = {
  // … C3 adds e.g. colorPrimary: '--tss-color-primary'
} as const;

export type TssClassName = (typeof tssClass)[keyof typeof tssClass];
export type TssVariableName = (typeof tssVar)[keyof typeof tssVar];

/** Gets a `var(--tss-…)` reference, optionally with a fallback that is itself a catalogue variable. */
export function cssVar(name: TssVariableName, fallback?: TssVariableName): string;
```

A unit test asserts every class starts with `tss-` and every variable with `--tss-` (D45). C3 adds a test that the docs list exactly the exported names.

## 12. Tests and contract guards

- **Runtime units** (`version`, `issues`, `catalogue`, `inclusion`): ordinary tests.
- **Type tests (`*.test-d.ts`)** in `module/`, `testrunner/`, `theme/`:
  - a minimal module / runner / theme set compiles against the constructor types via `satisfies`;
  - `verify` can't return both `ignored` and `issues` (`@ts-expect-error`); `UnitTests` narrows by `status`;
  - **principle 2 guard:** a recursive `AssertJsonSafe<T>` type (rejects functions, `Map`, `Set`, `Date` and class instances; allows readonly arrays and optional members) applied to `ShowcaseData` and `TestResultsUpdate`. A plain `JsonValue` assignability check can't work: interfaces have no index signature, readonly arrays aren't `JsonValue[]`, and optional members include `undefined`.
- **Compat guard (D13, D80):** `src/compat/v0.1.test-d.ts` holds frozen minimal implementations written against the current contract. A change that would break them (a new required member, a narrowed type) fails CI. Before 1.0, a deliberate breaking change bumps the `-api` minor and replaces the guard file (`v0.2.test-d.ts`); from 1.0, files for later minors are added next to it and never edited (additive-only).
- **README samples:** every unit README carries a type-checked sample (D56); the `module/` README sample is a complete, minimal no-op module.

## 13. Documentation

- Each unit README documents its unit (the single place for that fact, D56).
- The `-api` package README: purpose, the export list, the versioning rules (§2), and a "where to start" for module, runner and theme authors. The full authoring guides are C13.
- `CONTRIBUTE.md` glossary gains: group, analysis, issue, rule, inclusion rules, route target, theme set.
- AGENTS.md trigger table: "changed a contract" → update the unit README; after the first publish, add an `-api` changeset (minor for breaking changes before 1.0) and update the compat guard (§12).

## Acceptance criteria

1. `npm run ci` passes with the `-api` units, their READMEs, runtime tests and type tests.
2. `-api`'s only `dependency` is `@ofzza/tsschema`.
3. The README samples type-check: the minimal module (a valid `ShowcaseModuleConstructor`, `satisfies` included), the minimal runner, the minimal theme set and the composition example.
4. The compat guard is a permanent negative type test: a `@ts-expect-error` fixture proves that adding a required member to `ShowcaseModule` would break the frozen implementations.
5. `AssertJsonSafe<ShowcaseData>` and `AssertJsonSafe<TestResultsUpdate>` hold, and a negative test proves the guard rejects a function, a `Map` and a `Date`.
6. Runtime tests: `IMPLEMENTED_API_VERSION` equals `-api`'s `package.json` version; `issue.error` / `issue.warning` produce the documented shape; the inclusion constants have their documented values; every catalogue class starts with `tss-` and every variable with `--tss-`.

## Recorded for later chunks

- **C9:** the `check --format json` report shape (`reportVersion`, tool info, summary, repo-level issues, the directory tree with per-node status and issues), exported as types from the CLI package and documented in the CLI README. The round-1 sketch is in this file's git history.
- **C3:** theme CSS is written unscoped and the build scopes it (D65); C3 designs the mechanics, the selector unscoped files are written against, and how `-base-vite` (C15) applies it. Theme extension is composition (D65), so C3 needs no extension mechanism, only docs for the pattern.
- **C11:** writing `BuildResult.files` (D64): path validation (relative, no `..`), the checkable inclusion rules (`index.html` present, no absolute build-machine paths), the output guard, an atomic swap into `outDir`, the marker file, and the written-files summary.
- **C15:** `-base-vite` builds with `build.write: false` and maps the Rollup output (chunks and assets) to `BuildFile[]`.
- **C5:** text docs (`Doc.format: 'text'`, D71) render as escaped `<pre>` with a catalogue class; link mapping per D73; heading collection into `Doc.headings`.
- **C6:** `loadTestRunner` with `candidates`, throw on ambiguity, `undefined` when nothing matches (D74); `optionFlags` parsing, precedence, collisions and `--help` (D79).
- **C11:** `check` before `build`/`dev`, `--no-check` / `check: false` (D76); group README lookup (`README.md` in group directories, D78); `onAnalysisUpdate` delivery to watch-mode `runTests` (D75).
- **C6:** per-package options (D69): merge each package's `moduleOptions` over the root ones and validate every merged result against `optionsSchema`.
- **C6:** the config (§10, D66) moves into the CLI package: `config.schema.json` / `package-config.schema.json` with sidecars, the inferred `TsshowcaseConfig` / `PackageConfig` types, `defineConfig`, the `exports` subpaths, and docs in the CLI README. Acceptance criteria carried over from C2: `dist` contains every `*.d.json.ts` sidecar next to its `.json`, and a consumer-style type test over `dist` infers `TsshowcaseConfig` as an object type, not `never`; the published `config.schema.json` validates a sample `tsshowcase.config.json` with ajv and rejects an invalid one. Also: ajv (draft 2020-12, `useDefaults`) for config and module options; readable error formatting.
- **Before `1.0.0` (after C14):** tsschema record/tuple inference must have landed, and the `unknown` gap in §10 closed.
- **C14:** first publish is `0.1.0` (D80); the PR changeset check is switched on after it; PUBLISH.md documents the `0.x` major-changeset gotcha; whether `0.x` releases also use a `next` dist-tag is decided there. The pack-and-install smoke test also checks the sidecars reach the tarball and that a consumer's `defineConfig` call type-checks.

## Out of scope

Loading and version checks (C6), the catalogue contents (C3), any implementation of the contracts (C4–C11), authoring guides (C13).

## Open questions

- ✅ **Q1 Schema technology:** JSON Schema (draft 2020-12) for module options and config, ajv for validation in the CLI, `@ofzza/tsschema` to infer the TS types. Record/tuple inference is on tsschema's roadmap; temporary `unknown`s are acceptable until `1.0.0` (§10, D80). Round 3: the schemas are plain `.json` files with tsschema sidecars, not `as const` literals (D58, D62).
- ✅ **Q2 Showcases per unit:** an array (`showcases[]`).
- ✅ **Q3 Per-package config in workspaces:** per-package config files (the user's preference, familiar from tsconfig/.gitignore). Only in workspace package directories in v1, with a restricted `PackageConfig` schema; repo-wide keys there are an error. Nested directory configs could be added later, additively.
- ✅ **Q4 `CheckReport`:** a CLI detail, not in `-api`. `-api` holds only the internal plugin contracts; the report is external output, consumed only by the CLI, documented in the CLI README, with a `reportVersion` field. Specified in C9.
- ✅ **Q5 Required methods (D63):** only `analyze` is required; everything else is optional and feature-detected per action, including `verify`. The user's argument: a stub `verify` is as bad as a fake `build`.
- ✅ **Q6 Dev data updates (D68):** full snapshots, with an increasing `revision` on snapshots and test updates so stale messages can be dropped. Patches can be added later, additively.
- ✅ **Q7 `build` and `check` (D76):** `check` is a prerequisite of `build` and `dev`; check errors fail `build` by default (`--no-check` opts out); `dev` shows issues but never fails on them. (Claude first proposed the opposite; the user's question turned it around.)
- ✅ **Q8 Severity overrides applied by the CLI** (D70), which derives an issue's package when unset.
- ✅ **Q9 `resolveRoute` signature (D67):** `resolveRoute(target, ctx, heading?)` with the tagged `RouteTarget`; routes stored in the data.
- ✅ **Q10 Versioning (D80):** every package starts at `0.1.0`; no changesets before the first publish; pre-1.0, minors may break; additive-only from `1.0.0`. (The user's revision of the `1.0.0` proposal.)
