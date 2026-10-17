# Chunk plan C2: Plugin contracts (`@ofzza/tsshowcase-api`)

> Status: **DRAFT, round 3** (2026-10-17). Decisions referenced as `D<n>` are in AGENTS.md §3. Stays open until the C3 plan is approved (C3 fills the catalogue slot, D28).

## Goal

The complete, documented v1 contracts between the CLI and its plugins (rendering modules, test runners, themes), plus the shared data model, issue model, config schema and output inclusion rules. Everything lives in `@ofzza/tsshowcase-api`: TypeScript types, a handful of runtime constants and tiny helpers, no heavy dependencies.

The sketches below are the **proposal under review**. They use real TypeScript so they can be argued about precisely. JSDoc is left out of the sketches for brevity; the implementation documents every member (D10).

## Design principles

1. **Additive-only within a major (D13).** Every member added after 1.0 is optional and feature-detected. Every input and output object may gain fields. Consumers on both sides ignore fields they don't know.
2. **Plain data across the boundary.** Everything the CLI hands to `build`/`dev` (the `ShowcaseData`) is JSON-serialisable: no functions, no `Map`s, no class instances, no `Date`s (ISO strings instead). Functions only appear on _context_ objects (`read()`, `loadTestRunner()`, subscriptions), never in data.
3. **Repo-relative POSIX paths everywhere in data.** Absolute paths appear only on contexts (`repo.root`, `outDir`). This keeps the user's filesystem layout out of the published showcase, and makes the data identical on Windows and Linux.
4. **Directories are identities.** A package is identified by its directory (`'.'` for the root), a unit by its directory (D36: one unit per directory). No separate id scheme to keep in sync.
5. **Rule violations are data, failures are exceptions.** `verify`/`analyze` return issues (D2). A module throws only for genuine failures (bugs, bad config, a crashed tool); the CLI catches, names the module, and reports.
6. **The CLI owns cross-cutting behaviour once,** so each module doesn't reimplement it: severity overrides, file writing for `init`, output-dir guarding, file watching, markdown rendering, theme resolution.
7. **Every method may be async** (`MaybePromise<T>`), except `resolveRoute`, which the markdown pipeline calls for every link and must stay synchronous.

## 1. Package layout (dogfooding D41)

Each exported unit is its own directory with a README (D41). Type-only units need only a README (D38); units with runtime code also get tests. Contracts are additionally locked by type tests (§12).

```
packages/tsshowcase-api/src/
  index.ts                 re-exports every unit
  version/                 IMPLEMENTED_API_VERSION                                 runtime
  common/                  MaybePromise, Logger, JsonValue, Unsubscribe            types
  schema/                  JsonSchema type + OptionsOf<TSchema> (tsschema-based, Q1) types
  config/*.schema.json     the config JSON Schemas themselves, plain JSON + committed sidecars (§10)
  data/                    the data model: repo, packages, units, docs, tests      types
  issues/                  Issue, Severity, RuleDescriptor + issue() helpers       runtime
  module/                  ShowcaseModule + its contexts                           types
  testrunner/              TestRunner + raw test result hierarchy                  types
  theme/                   ThemeSet, ThemeDefinition                               types
  config/                  TsshowcaseConfig + defineConfig()                       runtime
  catalogue/               class + CSS variable constants (slot; C3 fills it)      runtime
  inclusion/               output inclusion constants (theme attribute, storage key) runtime
```

`-api`'s only dependency is **`@ofzza/tsschema`**, used for its types (its public declarations reference them, so it must be a real dependency, not a devDependency). It does not depend on `semver` (C6's version check lives in the CLI) or on a validator: **ajv lives in the CLI** (Q1).

## 2. Versioning (D13, D31)

- `IMPLEMENTED_API_VERSION` is read from `-api`'s own `package.json` (`import pkg from '../../package.json' with { type: 'json' }`), so there is exactly one place where the interface version lives.
- **C2 sets the `-api` version to `1.0.0`.** It stays unpublished until C14. Contract changes made before the first publish are edited in place and don't bump the version; the first published version is `1.0.0`.
- **The interface changelog is `-api`'s changesets `CHANGELOG.md`** (D31: the `-api` version _is_ the interface version, so its changelog _is_ the interface changelog). One place per fact (D56). Members added after 1.0 also say "Added in 1.x." in their JSDoc prose (the code style allows only `@param`/`@returns`, so there is no `@since`).
- Every plugin class declares `static readonly apiVersion: string`, a semver range (e.g. `'^1.0'`). The CLI checks `satisfies(IMPLEMENTED_API_VERSION, apiVersion)` at load time (C6).

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
  readonly generatedAt: string; // ISO timestamp
  readonly repo: RepoNode;
  readonly layout: 'single' | 'workspace';
  readonly packages: readonly PackageNode[]; // never includes a private workspace root
  readonly meta?: { readonly [key: string]: JsonValue }; // module-private extension bag
}

export interface RepoNode {
  readonly name: string; // root package name, or the directory name
  readonly info?: PackageInfo; // root manifest, if any
  readonly readme?: Doc; // root README.md
  readonly licence?: Doc; // root LICEN[CS]E[.md|.txt] (D12)
  readonly vcs?: VcsInfo;
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
  readonly markdown: string; // raw (D23)
  readonly html: string; // CLI-rendered (C5): catalogue classes, heading ids, rewritten links
  readonly headings: readonly DocHeading[]; // for tables of contents
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
  readonly children: readonly HierarchyNode[];
}

export interface UnitNode {
  readonly type: 'unit';
  readonly dir: RepoPath; // identity
  readonly name: string; // display name (module-chosen, defaults to the directory name)
  readonly kind: UnitKind; // module-chosen summary kind (D37)
  readonly exports: readonly ExportedSymbol[];
  readonly readme?: Doc; // optional in data: a missing README is an issue, not a missing page
  readonly showcases: readonly ShowcaseEntry[]; // may be empty (D37); see Q2
  readonly tests: UnitTests;
  readonly children: readonly HierarchyNode[];
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

`TestFileResult` is the runner's raw shape (§6). A unit with no test files is `available` with zero files; whether that is acceptable is a `verify` concern (D37/D38), not a data concern.

### 4.3 Root vs package docs, and the single-package collapse

- `repo.readme` / `repo.licence` are the root files. `package.readme` / `package.licence` are each package's own files.
- **Single-package repo** (`layout: 'single'`): `packages[0].dir === '.'`, and its `readme`/`licence` are the _same_ docs as the repo's. Modules must collapse the package level (D19): home shows the README once, units hang directly under home.
- **Workspace** (`layout: 'workspace'`): the workspace root is never a package (even if it isn't `private`). The home page shows `repo.readme`.
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
  new (init: ModuleInit<TOptions>): ShowcaseModule;
}

export interface ModuleInit<TOptions> extends PluginHost {
  readonly options: TOptions; // validated against optionsSchema, defaults applied by the schema
  readonly repoRoot: string; // absolute
}
```

The CLI constructs **one instance per CLI invocation** and calls its methods in the order of §5.9. Methods may be called concurrently (e.g. `verify` for many directories at once), so a module must not keep per-call state on `this`.

### 5.2 Methods: which are required

```ts
export interface ShowcaseModule {
  // Required in 1.0
  analyze(ctx: AnalyzeContext): MaybePromise<AnalysisResult>;
  verify(dir: DirectoryInput, ctx: VerifyContext): MaybePromise<VerifyResult>;

  // Optional in 1.0, feature-detected (the CLI reports a clear "module X does not support Y")
  resolveRoute?(target: RouteTarget, heading?: string): string; // required for build/dev
  runTests?(ctx: TestContext): MaybePromise<TestResults>;
  build?(ctx: BuildContext): MaybePromise<BuildResult>;
  dev?(ctx: DevContext): MaybePromise<DevSession>;
  scaffold?(ctx: ScaffoldContext): MaybePromise<ScaffoldResult>;
  dispose?(): MaybePromise<void>;
}
```

Rationale: `check` is the one action every module must support (the assignment's core). A check-only module (e.g. a pure "conventions linter" for a framework with no showcase app yet) is legitimate. `build` without `resolveRoute` is rejected at load time for the `build`/`dev` actions.

### 5.3 Shared context pieces

```ts
/** Lazy, cached, read-only view of the repo (D40). Paths are repo-relative. The CLI records reads for dev invalidation. */
export interface RepoFiles {
  read(path: RepoPath): Promise<string>; // utf-8; rejects if missing
  readJson<TValue = unknown>(path: RepoPath): Promise<TValue>;
  exists(path: RepoPath): Promise<boolean>;
  list(dir: RepoPath): Promise<DirectoryListing>; // non-ignored entries only
}

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
}
```

### 5.4 `analyze` (D5) and repo-level verification

Runs once per `check`/`build`/`dev` (and again in `dev` when files are added or removed). Typically uses the TS compiler API against the real filesystem (`repo.root`), so its reads are not tracked through `RepoFiles`; that is fine because `dev` re-analyses on structural changes anyway (C11).

```ts
export interface AnalyzeContext {
  readonly repo: RepoInput;
  readonly signal: AbortSignal; // dev aborts a stale analysis
}

export interface AnalysisResult {
  readonly packages: readonly PackageAnalysis[]; // one per analysed package; a package may be omitted (no entry, not applicable)
  readonly issues?: readonly Issue[]; // repo-/package-level issues (e.g. unresolvable entry, export whose source is shared)
  readonly state?: unknown; // opaque, handed back to verify/runTests/scaffold; never serialised
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
  readonly readme?: RepoPath; // where the module expects/found the README; CLI reads and renders it
  readonly showcases?: readonly ShowcaseEntry[];
  readonly meta?: { readonly [key: string]: JsonValue };
}
```

**Repo-level verification is covered by `analyze` returning issues**, rather than a separate hook. Each issue may carry a `dir` (attached to that directory in the report) or none (shown at repo/package level, using `package` when set). One pass, no duplicated analysis.

### 5.5 `verify` (D2, D40)

Called for **every non-ignored directory** of the repo (requirement), after `analyze`.

```ts
export interface DirectoryInput {
  readonly path: RepoPath;
  readonly package?: RepoPath; // the nearest package directory that owns it, if any
  readonly files: readonly string[]; // names in this directory, non-ignored
  readonly directories: readonly string[]; // names of non-ignored subdirectories
  read(file: string): Promise<string>; // lazy and cached (D40); relative to this directory
  exists(file: string): Promise<boolean>;
}

export interface VerifyContext {
  readonly repo: RepoInput;
  readonly analysis: AnalysisResult;
  readonly unit?: AnalyzedUnit; // the unit at this directory, if analyze found one
}

export type VerifyResult = { readonly ignored: true } | { readonly ignored?: false; readonly issues: readonly Issue[] };
```

An ignored directory still shows in the report tree (greyed out, "ignored by module"), since the requirement asks for the full non-ignored structure. Only `.gitignore` + config `exclude` remove a directory from the tree entirely.

### 5.6 `resolveRoute` (D35)

```ts
export type RouteTarget =
  | { readonly type: 'home' }
  | { readonly type: 'package'; readonly package: RepoPath }
  | { readonly type: 'unit'; readonly package: RepoPath; readonly unit: RepoPath };

// resolveRoute(target, heading?) → an in-app URL relative to the app's index.html, e.g. '#/button' or '#/button?h=props'
```

- `heading` is a heading id from `Doc.headings` (C5 generates the ids, so the module and the markdown pipeline agree).
- The returned URL must work under any base path (D43 inclusion rules): relative, no leading `/`.
- **Purity:** the result depends only on its arguments plus what the module saw in `analyze`. The CLI calls it while rendering markdown, before `build`.

### 5.7 `runTests` (D3, D34, D42, D18)

```ts
export interface TestContext {
  readonly repo: RepoInput;
  readonly analysis: AnalysisResult;
  readonly mode: 'run' | 'watch';
  readonly signal: AbortSignal; // in watch mode, aborting stops watching
  loadTestRunner(ref?: string): Promise<LoadedTestRunner | undefined>; // explicit ref, else auto-detection (D42); undefined = none found
  onTestUpdate(update: TestResultsUpdate): void; // watch mode only: push mapped updates to the CLI
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
  readonly units: { readonly [unitDir: RepoPath]: UnitTests }; // only the units that changed; full replacement per unit
}
```

In `watch` mode `runTests` resolves with the first complete result and keeps calling `onTestUpdate` until `signal` aborts.

### 5.8 `build` and `dev` (D43)

```ts
export interface BuildContext {
  readonly data: ShowcaseData;
  readonly themes: readonly ResolvedTheme[]; // in config order
  readonly defaultTheme?: string; // explicit config choice; otherwise the inclusion rules decide (§8)
  readonly outDir: string; // absolute; the CLI has already guarded it (C11)
  readonly repoRoot: string; // absolute; to turn RepoPaths (showcase files) into imports
  readonly cacheDir: string; // absolute, module-private scratch space, persists across runs
  readonly signal: AbortSignal;
}

export interface BuildResult {
  readonly files?: readonly string[]; // written files, relative to outDir, for the CLI's summary
}

export interface DevContext extends Omit<BuildContext, 'outDir'> {
  readonly host?: string;
  readonly port?: number;
  onDataUpdate(listener: (data: ShowcaseData) => void): Unsubscribe; // full snapshot each time (Q6)
  onTestUpdate(listener: (update: TestResultsUpdate) => void): Unsubscribe;
}

export interface DevSession {
  readonly url: string;
  close(): Promise<void>;
}
```

- `data` already contains test results (or `not-run`/`unavailable`), README HTML and the tree.
- In `dev`, test updates reach the dev server separately (`onTestUpdate`) so a test run doesn't force a full data push. The CLI also merges them into its own copy, so the next full `onDataUpdate` snapshot is consistent.
- Theme CSS edits in `dev` are the module's dev server's job (it watches the `css` files it was given). Changing which themes are configured needs a restart.

### 5.9 Call order per action

| Action  | Calls                                                                                                                                                                 |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `check` | `analyze` → `verify` × every directory → severity overrides applied by the CLI → report                                                                               |
| `init`  | `analyze` → `scaffold` → the CLI writes the returned files                                                                                                            |
| `build` | `analyze` → `runTests('run')` (unless `--no-tests`) → CLI reads READMEs and renders HTML via `resolveRoute` → `build`. `check` is **not** implied; see Q7              |
| `dev`   | as `build`, but `runTests('watch')` and `dev`; on file changes the CLI re-renders / re-analyses and pushes `onDataUpdate`; test updates flow via `onTestUpdate` (D42) |
| all     | `dispose` at the end, also on Ctrl-C                                                                                                                                  |

### 5.10 `scaffold` (D16)

The module plans the files; the **CLI writes them** (one implementation of dry-run, never-overwrite-silently and reporting). C8 decides `init` semantics (new unit vs filling gaps, adding the entry export).

```ts
export interface ScaffoldContext {
  readonly repo: RepoInput;
  readonly analysis: AnalysisResult;
  readonly target: RepoPath; // the directory passed to `tsshowcase init <dir>`
  readonly args: { readonly [key: string]: string | boolean }; // extra `init` flags (e.g. --kind hook), module-defined
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

Validated by the CLI with ajv against `optionsSchema` (Q1), with schema `default`s applied (`useDefaults`). Without a schema the options are passed through as given. Authors derive their options type from the schema rather than writing it twice. **Our own modules keep the schema as a plain `options.schema.json` with a tsschema sidecar** (the same convention as the config, §10); third-party authors may do the same or write an `as const` literal; the contract only needs the value.

```ts
import optionsSchema from './options.schema.json' with { type: 'json' }; // typed by the committed options.schema.d.json.ts sidecar
type Options = OptionsOf<typeof optionsSchema>; // via tsschema's TSSchemaType: e.g. { testRunner?: string }
export default class MyModule implements ShowcaseModule {
  static readonly apiVersion = '^1.0';
  static readonly optionsSchema = optionsSchema;
  constructor(init: ModuleInit<Options>) {}
  // …
}
```

 `testRunner` (D34) is a **module option by convention**: modules that run tests should accept `testRunner?: string` and pass it to `ctx.loadTestRunner()`.

### 5.12 Room for the playground (D25)

Nothing in 1.0. A later minor adds an optional `playground?` field on `UnitNode` / `ShowcaseEntry` and, if needed, an optional module method. Existing modules ignore it.

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
  new (host: PluginHost): ThemeSet;
}

export interface ThemeSet {
  readonly themes: readonly ThemeDefinition[];
}

export interface ThemeDefinition {
  readonly name: string; // unique within a build; the value of the theme attribute (§8)
  readonly label?: string; // shown in the switcher; defaults to name
  readonly scheme: 'light' | 'dark'; // drives the prefers-color-scheme default and `color-scheme` (D7, D44)
  readonly css: readonly string[]; // absolute paths to the theme's CSS files, in order
  readonly extends?: string; // another theme's name; C3 defines the semantics
}

export interface ResolvedTheme extends ThemeDefinition {
  readonly reference: string; // the config reference it came from
}
```

**Reference syntax:** `<ref>` selects every theme in the set; `<ref>#<name>` selects one. Built-in names (`default-light`, `default-dark`) are aliases the CLI maps to `@ofzza/tsshowcase-theme-default#default-light` etc.

C3 decides how the CSS is written (whether files carry their own `[data-theme=…]` scoping or the build wraps them), how `extends` works, and everything about the variable hierarchy.

## 8. Output inclusion rules (`inclusion/`, D43)

Binding on **every** module's output, whether or not it uses `-base-vite`. Documented in the `inclusion/` README; constants exported so nobody hardcodes the strings.

1. **Static and relocatable.** `build` writes a self-contained static site with `index.html` at `outDir`'s root. It works when served from any sub-path (GitHub Pages project sites): all URLs relative, routing needs no server rewrites (hash routing or equivalent).
2. **No leaks.** No absolute filesystem paths from the build machine appear in the output. No runtime requests to third-party origins (fonts, CDNs, analytics) unless the user's own showcase code makes them.
3. **Every configured theme is included** and applied by setting `THEME_ATTRIBUTE` (`'data-theme'`) to the theme's `name` on the `<html>` element (C3 confirms the element and scoping).
4. **Initial theme selection,** applied before first paint (no flash): the stored choice (`localStorage[THEME_STORAGE_KEY]`, `'tsshowcase:theme'`) if it names an included theme → else the configured `defaultTheme` → else the first theme whose `scheme` matches `prefers-color-scheme` → else the first theme. With only one theme, no switcher is rendered.
5. **The switcher** stores the user's choice under `THEME_STORAGE_KEY` and uses the catalogue's switcher class (C3).
6. **README HTML is inserted verbatim** inside an element carrying the catalogue's content region class (`tss-content`, D44), so links, ids and theming stay intact.
7. **Showcases are rendered without shell styling leaking in** (D44: no bare-element styling outside regions). How each framework isolates them is the module's business.

## 9. Issue model (`issues/`)

```ts
export type Severity = 'error' | 'warning';

export interface Issue {
  readonly rule: string; // namespaced id, e.g. 'react/readme-required', 'headless/render-pure'
  readonly severity: Severity; // the module's default; the CLI applies config overrides (below)
  readonly message: string; // one sentence, no trailing period
  readonly dir?: RepoPath; // set for analyze issues that belong to a directory; verify issues get it from context
  readonly package?: RepoPath;
  readonly file?: RepoPath;
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
- `check` exits non-zero when any `error` remains after overrides (D16).

## 10. Config schema (`config/`, D6)

**The JSON Schema is the source of truth (D58).** The schemas are **plain JSON files**, `config/config.schema.json` and `config/package-config.schema.json`, each with a committed tsschema sidecar (`*.d.json.ts`, `tsschema generate --mode sidecar`) that gives the JSON import its literal types. `-api` exports them as values (`configSchema`, `packageConfigSchema`), and the TS types below are **inferred** from them with tsschema, never written by hand. Documentation lives in each property's `description` (JSON has no comments), which editors show on hover. The shapes the inferred types must have:

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
  exclude?: boolean; // leave the package out of check and build entirely
  rules?: { [ruleId: string]: Severity | 'off' }; // merged over the root rules
  moduleOptions?: { [key: string]: unknown }; // shallow-merged over the root moduleOptions
}

export function defineConfig(config: TsshowcaseConfig): TsshowcaseConfig; // identity, for typing in tsshowcase.config.ts
```

- CLI flags override file values (D6). C6 validates config files with ajv against these schemas; normalisation (e.g. `themes: string` → `[string]`) and checks JSON Schema can't express (valid semver ranges, resolvable references) happen in code after validation.
- The JSON files are **published as they are**, as extra `exports` subpaths (`@ofzza/tsshowcase-api/config.schema.json`, `…/package-config.schema.json`), so `tsshowcase.config.json` gets editor completion via `$schema`, and the CLI README links them. No emit step.
- **Sidecar gotcha (verified by experiment, TS 6.0.3):** `tsc` copies the `.json` into `dist` (when `src/**/*.json` is in the build `include`) but **not** the `.d.json.ts` sidecar. Without the sidecar, consumers' inferred `TsshowcaseConfig` silently becomes `never`. The C1 build therefore copies sidecars into `dist` (C1 amendment, D62). With the sidecar present, consumers need no `allowArbitraryExtensions` of their own.
- **Known gap (temporary):** until tsschema infers records (`additionalProperties`) and tuples, map-shaped fields (`rules`, `moduleOptions`) infer as `{ [key: string]: unknown }`. We accept that: the schemas (and so runtime validation) are already exact, and tightening an inferred _input_ type later only rejects configs that ajv already rejects at runtime. Code that reads those maps (C6, C8) narrows at the point of use. Must be resolved before the v1.0.0 release (C14).

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

- **Runtime units** (`version`, `issues`, `config`, `catalogue`, `inclusion`): ordinary tests.
- **Type tests (`*.test-d.ts`)** in `module/`, `testrunner/`, `theme/`:
  - a minimal module / runner / theme set compiles against the constructor types via `satisfies`;
  - `verify` can't return both `ignored` and `issues`; `UnitTests` narrows by `status`; `ShowcaseData` is assignable to `JsonValue` (guards principle 2).
- **Additive-only guard (D13):** `src/compat/v1.0.test-d.ts` holds frozen minimal implementations written against 1.0. A later change that would break them (a new required member, a narrowed type) fails CI. Files for later minors are added next to it and never edited.
- **README samples:** every unit README carries a type-checked sample (D56); the `module/` README sample is a complete, minimal no-op module.

## 13. Documentation

- Each unit README documents its unit (the single place for that fact, D56).
- The `-api` package README: purpose, the export list, the versioning rules (§2), and a "where to start" for module, runner and theme authors. The full authoring guides are C13.
- `CONTRIBUTE.md` glossary gains: group, analysis, issue, rule, inclusion rules, route target, theme set.
- AGENTS.md trigger table: "changed a contract" → update the unit README, add a changeset for `-api`, add a compat test file if released.

## Acceptance criteria

1. `npm run ci` passes with the `-api` units, their READMEs, runtime tests and type tests.
2. `-api`'s only `dependency` is `@ofzza/tsschema`.
3. The README sample no-op module type-checks and is a valid `ShowcaseModuleConstructor`.
4. The compat guard fails when a required member is added to `ShowcaseModule` (checked once by hand, reverted).
5. `ShowcaseData` is `JsonValue`-assignable (type test).
6. `dist` contains every `*.d.json.ts` sidecar next to its `.json` (test over the built output), and a consumer-style type test over `dist` infers `TsshowcaseConfig` as an object type, not `never`.
7. The published `config.schema.json` validates a sample `tsshowcase.config.json` with ajv, and rejects an invalid one (test; ajv is a root devDependency for this).

## Recorded for later chunks

- **C9:** the `check --format json` report shape (`reportVersion`, tool info, summary, repo-level issues, the directory tree with per-node status and issues), exported as types from the CLI package and documented in the CLI README. The round-1 sketch is in this file's git history.
- **C6:** ajv (draft 2020-12, `useDefaults`) for config and module options; readable error formatting.
- **C14:** tsschema record/tuple inference must have landed, and the `unknown` gap in §10 closed. The pack-and-install smoke test also checks the sidecars reach the tarball and that a consumer's `defineConfig` call type-checks.

## Out of scope

Loading and version checks (C6), the catalogue contents (C3), any implementation of the contracts (C4–C11), authoring guides (C13).

## Open questions (round 1)

- ✅ **Q1 Schema technology:** JSON Schema (draft 2020-12) for module options and config, ajv for validation in the CLI, `@ofzza/tsschema` to infer the TS types. Record/tuple inference is on tsschema's roadmap; temporary `unknown`s are acceptable until the v1.0.0 release (§10). Round 3: the schemas are plain `.json` files with tsschema sidecars, not `as const` literals (D58, D62).
- ✅ **Q2 Showcases per unit:** an array (`showcases[]`).
- ✅ **Q3 Per-package config in workspaces:** per-package config files (the user's preference, familiar from tsconfig/.gitignore). Only in workspace package directories in v1, with a restricted `PackageConfig` schema; repo-wide keys there are an error. Nested directory configs could be added later, additively.
- ✅ **Q4 `CheckReport`:** a CLI detail, not in `-api`. `-api` holds only the internal plugin contracts; the report is external output, consumed only by the CLI, documented in the CLI README, with a `reportVersion` field. Specified in C9.
- **Q5 Required methods:** only `analyze` + `verify` required, everything else feature-detected (proposed), vs requiring the full set.
- **Q6 Dev data updates:** full snapshots in 1.0 (proposed; simple, data is small), vs patches.
- **Q7 Does `build` imply `check`?** Proposal: no; `build --check` opt-in to fail on check errors.
- **Q8 Severity overrides applied by the CLI** (proposed) vs passed to the module.
- **Q9 `resolveRoute` signature:** D35 says `resolveRoute(target, heading?)`; the proposal keeps that and defines `target` as a tagged object holding directories.
- **Q10 `-api` set to version `1.0.0`** in C2, with pre-release edits in place (proposed).
