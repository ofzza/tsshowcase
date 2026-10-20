# Chunk plan C2: Plugin contracts (`@ofzza/tsshowcase-api`)

> Status: **APPROVED** (round 5, 2026-10-18; review triage in `02a-c2-plan-review.md`). Amendment B1 from C3 planning (catalogue filled, theme stylesheets, isolation; D81–D90) **re-approved** with the C3 plan (2026-10-18). The catalogue slot is filled; the plan is closed. Editorial corrections from the checkpoint review (`01b-checkpoint-review.md`: `check` and `outDir` in the config sketch, the dev stylesheet bullet, inclusion constants, wording) and two decisions from it (D92 bare elements only inside `.tss-content`, D93 `build [outDir]`) were applied on 2026-10-18 with the user's approval. **Amendment B2 (prototype spikes, `01c-spike-results.md`) re-approved 2026-10-18:** test-runner contract additions (`project`, `unhandledErrors`, `repoRoot`, substring filters, process restore, no-tests = completed, best-effort `removed`) and rule 7's `!important`/setup-option/stacking/event notes. **Amendment B3 (review before wave P3, `01d-pre-p3-review.md`) re-approved 2026-10-18 with super-plan R4:** rule 7, shell-mode showcases get no scheme selector (D100); §5.5 and §10, `include` and the marker-file skip (D101); §5.8, `BuildContext.themes` narrowed to name, label and scheme (D102). **Amendment B4 (C4 planning, `04a-c4-plan-review.md`) re-approved 2026-10-18 with the C4 plan and super-plan R5:** §4 `VcsInfo` without `defaultBranch`, `{path}` the only template placeholder (D110); §5.3/§5.5 `read`/`exists`/`list` treat ignored paths as nonexistent unless `{ ignored: true }`, `.git`/`node_modules` never reachable, `DirectoryListing` of `DirectoryEntry` descriptors, lexical root boundary (D108); §5.4/§5.5 "every scanned directory" and the scan rules (D103, D109); §10 `include` → `scope`, `exclude` → `ignore`, `vcs` keys (D107, D110). **Amendment B5 (C6 planning, `06a-c6-plan-review.md`) re-approved 2026-10-19 with the C6 plan and super-plan R6:** §5.11 path- and reference-marked module options resolved per source by the CLI, `OPTION_FORMAT` in `module/` (D116); §5.7 relative `file:` references resolve against the repo root (D114, D116); §10 wording: the schemas live in `config-schema/` and are copied to the package root by the CLI's build (C6 §5). **Amendment B6 (C5 planning, `05a-c5-plan-review.md`) approved 2026-10-19 with the C5 plan and super-plan R7:** §4 `Doc.assets` / `DocAsset` (D120); §5.6 `heading` is any element id C5 produced or kept, link mapping refined (D121, D124, D128); §5.8 `DevContext.docAssetsDir` (D125), `BuildResult.files` excludes doc assets; §5.9 `check` reports `docs/*` issues (D123); §8 rules 1, 2 and 6 (reserved `tss-assets/`, README content exception, URLs relative to the site root), `DOC_ASSETS_DIR`. **Amendment B7 (C7 planning, `07a-c7-plan-review.md`) approved 2026-10-19 with the C7 plan and super-plan R8:** §6 `RunnerRepoInput` gains `layout`/`info?`, `TestRunOptions.repo` replaces `root`/`repoRoot` and `files` become repo-relative paths (D132), `RawTestReport.failedRoots` (D135), suite `error?`, `unhandledErrors` replacement semantics in updates, aborts resolve with `error`, the refined process rule (restore timing, removing the tool's process handlers); §4.2 `failedRoots` units are `unavailable`; §5.7 sample and wording, `testRunner` root-only (D136); §5.11 the same. **Amendment B8 (C15 planning, `15b-c15-plan-review.md` S7, `15c-c15-plan-review.md` T17) approved 2026-10-19 with the C15 plan:** §5.8 last bullet and §8 rule 3 (wording): in `dev` the module applies the updated stylesheets after `onThemeStylesheetsUpdate`, re-reading the files or using the pushed contents (`-base-vite` re-reads and hot-reloads them); §5.11 module options are root-only by default; per-package options are marked with the schema keyword `x-tsshowcase-per-package` (`PER_PACKAGE_OPTION_KEYWORD`, D148). **Amendment B9 (the representation redesign, `08b-representation-redesign.md`, D160–D199) **approved 2026-10-20** (shape decisions H1–H7, the agent review and its outcomes Q1–Q5, Fix 1–8, Skip 1–2 in `02b-c2-b9-review.md`):** design principles 2, 4, 6; §1 package layout under the new dogfood rules; §3 `JsonObject`; §4 data model replaced (`Representation`, `RepresentableNode`, `DocRef`, `Tests`, `TestsUpdate`, `RenderedDoc`; `ShowcaseData`, units, groups and `Doc` removed); §5.1 statics `kinds`, `testRunners`; §5.2 methods (`runTests` removed, `verify` per file and optional for `check`); §5.3 `ScannedTree`, `RepoInput`/`PackageInput` revised; §5.4 `AnalysisResult` per file; §5.5 `FileInput`/`VerifyContext`; §5.6 `RouteTarget` and link resolution; §5.7 removed; §5.8 `representation`, `docs`, `DataUpdate` (no separate docs update, 02b H2); §5.9 call order; §5.10 `ScaffoldContext`; §5.11 `testRunner` no longer an option; §6 `listTestFiles`/`TestListing`; §8 rules 6, 7 and the embedding note; §9 issue placement, rule ids, overrides, `--strict`; §10 config keys and directory configs; §12, §13, acceptance criteria, recorded points and open questions updated. Decisions referenced as `D<n>` are in AGENTS.md §3.

## Goal

The complete, documented v1 contracts between the CLI and its plugins (rendering modules, test runners, themes), plus the shared data model, issue model and output inclusion rules. (The user-facing config is a CLI detail, D66, §10.) Everything lives in `@ofzza/tsshowcase-api`: TypeScript types, a handful of runtime constants and tiny helpers, no heavy dependencies.

The sketches below are the **proposal under review**. They use real TypeScript so they can be argued about precisely. JSDoc is left out of the sketches for brevity; the implementation documents every member (D10).

## Design principles

1. **Additive-only within a major (D13).** Every member added after 1.0 is optional and feature-detected. Every input and output object may gain fields. Consumers on both sides ignore fields they don't know. Unions marked _open_ (`| (string & {})`) may gain values; consumers treat unknown values as "other".
2. **Plain data across the boundary.** Everything the CLI hands to `build`/`dev` as data (the `Representation`, §4, and `TestsUpdate`) is JSON-serialisable: no functions, no `Map`s, no class instances, no `Date`s (ISO strings instead). Functions only appear on _context_ objects (`read()`, `docs.render()`, subscriptions), never in data (B9). `DataUpdate` (§5.8) is an **envelope**, not data: its `representation` is the data and stays JSON-safe; `docs` and `state` travel alongside it as they do on the context.
3. **Repo-relative POSIX paths everywhere in data.** Absolute paths appear only on contexts and plugin outputs that never reach the browser (`repo.root`, `repoRoot`, `cacheDir`, `docAssetsDir`, `ThemeDefinition.css`, `BuildContext.themeStylesheets`). This keeps the user's filesystem layout out of the published showcase, and makes the data identical on Windows and Linux.
4. _(Removed by B9: "directories are identities". Nodes have opaque ids whose meaning comes from their `path` and `representation`, §4.1; packages are still identified by their directory.)_
5. **Rule violations are data, failures are exceptions.** The analysis and `verify` return issues (D166, D170, D177). A module throws only for genuine failures (bugs, bad config, a crashed tool); the CLI catches, names the module, and reports.
6. **The CLI owns cross-cutting behaviour once,** so each module doesn't reimplement it: the scan, associations, the representation and the check (D166, D168), the test runners (D172), link resolution (D183), severity overrides, output-dir guarding, file watching, markdown rendering, theme resolution. The module **enriches** (symbols, showcasing components, its own issues) and **presents** (D160 principle 3).
7. **The CLI performs all writes (D64).** Modules never write to the showcased repo or the output: `scaffold` and `build` _return_ files (paths + contents) and the CLI writes them. Reads should go through `RepoFiles` (D40) where practical; that is preferred, not enforced (the TS compiler in the analysis reads the disk directly). This is a contract rule, not a sandbox: module code could still write anywhere, but the correct path is the easy one.
8. **Every method may be async** (`MaybePromise<T>`), except `resolveRoute`, which the markdown pipeline calls for every link and must stay synchronous.

## 1. Package layout (dogfooding D41, B9)

`-api` follows the representation rules it defines (D161–D163; C1 amendment A4): each area is a directory with a `README.md`, which documents every exporting file in it (D163, strong association); each concept is one exporting file; files declaring **functions or classes** get a test named after them (`issue.ts` → `issue.test.ts`); files with only types or constants need only the README (D162), though the area's tests may still assert constant values (acceptance criterion 6). No per-directory barrels: `src/index.ts` re-exports the declaring files. Contracts are additionally locked by type tests (§12).

```
packages/tsshowcase-api/src/
  index.ts                 re-exports every declaring file
  version/                 IMPLEMENTED_API_VERSION                                         runtime
  common/                  MaybePromise, Logger, JsonValue, JsonObject, Unsubscribe, RepoPath types
  schema/                  JsonSchema type + OptionsOf<TSchema> (tsschema-based, Q1)       types
  data/                    the data model: Representation, nodes, docs, tests (§4)        types
  issues/                  Issue, Severity, RuleDescriptor + issue() helpers               runtime
  module/                  ShowcaseModule + its contexts, the scanned tree, OPTION_FORMAT, PER_PACKAGE_OPTION_KEYWORD   types + runtime
  testrunner/              TestRunner, TestListing + raw test result hierarchy             types
  theme/                   ThemeSet, ThemeDefinition                                       types
  catalogue/               class + CSS variable constants (C3 fills it)                    runtime
  inclusion/               output inclusion constants (theme attribute, storage key, …)    runtime
  compat/                  frozen contract guards (type tests only, §12; exports nothing, no README)
```

Exact file names inside each area are an implementation detail; the rule above decides what each file needs.

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
export type JsonObject = { readonly [key: string]: JsonValue }; // B9: the `meta` passthrough type

/** Repo-relative, POSIX-separated path, no leading `./`; `'.'` is the repo root. Documentation-only alias. */
export type RepoPath = string;

export interface Logger {
  debug(message: string): void;
  info(message: string): void;
  warn(message: string): void;
  error(message: string): void;
}

/** A JSON Schema (draft 2020-12): tsschema's schema type, so both a sidecar-typed JSON import and an `as const` literal are assignable. */
export type JsonSchema = TSSchemaJSONSchema; // from @ofzza/tsschema (`schema/` area)

/** The options a module receives: inferred from its schema with tsschema, **after** ajv's `useDefaults`, so properties with a `default` are required. */
export type OptionsOf<TSchema extends JsonSchema> = TSSchemaType<TSchema> /* with defaulted properties made required */;

/** Passed to every plugin constructor. */
export interface PluginHost {
  readonly apiVersion: string; // IMPLEMENTED_API_VERSION of the running CLI
  readonly cliVersion: string;
  readonly logger: Logger; // prefixed with the plugin's name
}
```

## 4. Data model (`data/`, B9)

The model is **multi-package from day one** (D16). A single-package repo is one package whose directory is `'.'`. The **CLI** builds all of it (D166, D168): the module contributes enrichments (§5.4) and reads the result. Replaces the pre-B9 `ShowcaseData`, `RepoNode`, `PackageNode`, `HierarchyNode`/`GroupNode`/`UnitNode`, `UnitTests`, `ExportedSymbol`, `ShowcaseEntry` and `Doc` (D187).

```ts
export interface Representation {
  readonly apiVersion: string; // interface version that produced this data
  readonly revision: number; // monotonic, shared with TestsUpdate (D186); 0 for the first snapshot (so 0 in build), increasing per dev snapshot
  readonly generatedAt: string; // ISO timestamp of this snapshot's assembly (changes per dev snapshot); SOURCE_DATE_EPOCH wins, for reproducible builds
  readonly layout: 'single' | 'workspace';
  readonly home: RepresentableNode & {
    readonly name: string; // the root manifest's name, else the repo directory's name
    readonly info?: PackageInfo; // root manifest, if any
    readonly vcs?: VcsInfo;
  }; // id 'home:.'
  readonly packages: readonly PackageRepresentation[]; // every package (D193), in walk order of their directories; never the workspace root
  readonly tests: Tests; // §4.4
  readonly issues: readonly Issue[]; // issues no node fits (repo-level, §9, D180); never repeated on a node
  readonly meta?: JsonObject; // from AnalysisResult.meta
}

export interface PackageRepresentation {
  readonly dir: RepoPath; // identity; '.' in a single-package repo
  readonly info: PackageInfo;
  readonly home: RepresentableNode; // the package's root documents; the repo home itself in a single-package repo (same id 'home:.')
  readonly root: RepresentableNode; // id 'root:<dir>'; the package's node tree
}

export interface RepresentableNode {
  readonly id: string; // opaque: modules never parse it (meaning via path and representation)
  readonly representation: 'direct' | 'collection' | 'indirect' | 'root' | 'home' | (string & {}); // open union (Q5 in 02b: 'collection' added)
  readonly path: RepoPath; // the file (direct), the virtual '<dir>/<key>' (collection: no such file exists), or the directory (indirect, root, home)
  readonly configuredBy?: RepoPath; // root only: the directory whose directory config merged into it (D179)
  readonly name?: string; // from configuration; modules derive a default ('index.*' → its directory's name)
  readonly slug?: string; // from configuration; URL-safe, validated by the CLI (D188)
  readonly order?: number; // from configuration
  readonly route?: string; // from resolveRoute, build/dev only (§5.6, D67)
  readonly items: readonly RepresentedItem[];
  readonly docs: readonly DocRef[];
  readonly tests: readonly Association[];
  readonly showcases: readonly ShowcaseAssociation[];
  readonly issues: readonly Issue[]; // each issue lives in exactly one place (§9); homes carry only issues about their own docs
  readonly children: readonly RepresentableNode[]; // always [] for homes
  readonly meta?: JsonObject;
}

/** Well-known kinds (D162); modules declare their own (§5.1 `kinds`). Open union. */
export type SymbolKind = 'component' | 'hook' | 'function' | 'class' | 'constant' | 'type' | (string & {});

export type AssociationLevel = 'primary' | 'strong' | 'weak' | (string & {}); // open union; primary > strong > weak (D173)

export interface RepresentedItem {
  readonly file: RepoPath; // the declaring file (a collection member's real path)
  readonly name: string; // the declared name ('default' without an identifier)
  readonly kind: SymbolKind; // effective, after kind overrides (D189)
  readonly meta?: JsonObject; // FileSymbol.meta, verbatim
}

export interface Association {
  readonly file: RepoPath;
  readonly association: AssociationLevel;
  readonly ordinal: number; // index in this node's array of this resource type
}

export interface ShowcaseAssociation extends Association {
  readonly name: string; // the showcasing component's export name ('default' for the default export); identity = (file, name) (D195)
  readonly meta?: JsonObject; // ShowcasingComponent.meta, verbatim (e.g. the shell opt-in, a title)
}

export interface DocRef {
  readonly path: RepoPath;
  readonly id: string; // §4.2
  readonly format: 'markdown' | 'text'; // D71
  readonly role?: 'readme' | 'licence' | 'changelog' | 'other' | (string & {}); // §4.3
  readonly association?: AssociationLevel; // unset on homes
  readonly ordinal: number;
}

/** What DocsAccess.render returns (§5.8); never part of the data. */
export interface RenderedDoc {
  readonly html: string; // C5: catalogue classes, prefixed element ids, rewritten links, local media rewritten to doc assets; text docs: escaped <pre>
  readonly headings: readonly DocHeading[]; // empty for text docs
  readonly assets: readonly DocAsset[]; // D120; empty for text docs
}

export interface DocHeading {
  readonly depth: 1 | 2 | 3 | 4 | 5 | 6;
  readonly text: string;
  readonly id: string; // the prefixed element id ('<doc-id>--<slug>', D184)
}

export interface DocAsset {
  readonly source: RepoPath; // the repo file, e.g. 'packages/ui/src/Button/docs/states.png'
  readonly path: string; // site path relative to the site root: 'tss-assets/<first 16 hex of sha256>/<file name>' (DOC_ASSETS_DIR, §8 rule 1)
}

export interface VcsInfo {
  readonly kind: 'git';
  readonly host: 'github' | 'gitlab' | 'bitbucket' | 'other';
  readonly webUrl?: string; // e.g. https://github.com/ofzza/tsshowcase
  readonly fileUrlTemplate?: string; // resolved file-link template; `{path}` (relative to the repo root) is the only placeholder, e.g. 'https://github.com/o/r/blob/HEAD/{path}' (D110: links always use HEAD; no git)
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
```

### 4.1 Nodes and the tree (D178, D179)

- **Kinds of node:** a **direct** node per exporting file and a **collection** node per collection (D190), each holding its symbols and their primary and strong associations (a collection node's `path` is virtual, `<dir>/<key>`; its members are the distinct `items[].file` values); an **indirect** node per directory config that creates a node, holding what its `paths` list plus its directory's README; the package **root** (always `root:<package dir>`, path = the package directory), into which a directory config at the package directory, or at a single literal scope path, merges (`configuredBy`); a **home** per root (repo and package) holding root documents only.
- **Nesting:** a node's parent is the node whose path is the nearest ancestor path of its own; nodes without one are children of the package root; collections nest by their directory. The tree is sparse (no automatic intermediate levels). A symbol listed by several nodes appears in each (duplicates are the module's to present).
- **Ids:** opaque strings (`home:<dir>`, `root:<package dir>`, others path-based in v1); modules never parse them.
- **Order (D178):** the CLI emits every array in its final default order, and `ordinal` is the index in it. A direct node's items in declaration order; a configured node's items as listed (files by the first matching `paths` entry, one glob's matches in walk order, each file's symbols in declaration order); listed resources first, then walk order; children with `order` first (ascending), then walk order, a collection sorting as a file named `<key>`; homes before the tree. Modules may re-sort as a presentation choice.
- **Associations (D173):** `primary` (name matches, explicit listings, a configured node's own directory README), `strong` (a directory README for every exporting file there; a single exporting file's leftovers), `weak` (unmatched resources, attached to the nearest root or indirect node above). The check counts primary and strong only (D174); weak associations are for presentation (D181).
- **Coverage and per-node test summaries are not part of the contract** (D182): modules compute them from `items`, `path`, the tree, `tests` associations and `Representation.tests`.

### 4.2 Doc ids (D184)

`DocRef.id` is `doc-` plus the first 8 hex digits of the sha256 of the doc's repo path (`doc-3f2a9c1e`); docs of one representation whose prefixes collide take the shortest longer prefix, in steps of 4 hex digits, that makes them distinct. Every element id inside a rendered doc (headings, footnotes, author anchors) is prefixed `<doc-id>--`, so any number of docs can share a page; README authors still write GitHub-style `#props` fragments, which the CLI rewrites (§5.6).

### 4.3 Homes (D185)

- The repo `home` and each package's `home` hold the root's documents as `DocRef`s: every markdown file directly in that root, plus `README`, `LICEN[CS]E` and `CHANGELOG` without an extension (and `README`/`LICEN[CS]E` as `.txt`), case-insensitive, non-markdown as `format: 'text'`.
- **Roles:** `readme` (README forms, also on directory READMEs elsewhere), `licence` (`LICEN[CS]E` forms), `changelog` (`CHANGELOG`, any case, `.md` or no extension), `other` (the remaining home docs); other docs carry no role.
- **The module selects** what to show (`react-default`: README and licence by default, a `homeDocs` option for more).
- **Single-package repos** (`layout: 'single'`): `packages[0].dir === '.'` and `packages[0].home` is the repo home (same id). The module decides whether to merge the home with the package root (D19).
- **Workspaces:** the workspace root is never a package. **No manifest:** `packages: []`, `home.name` is the directory name, and nothing is in scope.
- **No inheritance in the data:** a workspace package without a licence simply has none in its home; a module may fall back to the repo home's and should say so on the page.

### 4.4 Tests in the data (D172, D186)

```ts
export interface Tests {
  readonly status: 'ran' | 'not-run' | 'unavailable' | (string & {}); // open union
  readonly reason?: '--no-tests' | 'no tests required' | 'check' | 'init' | 'no runner' | (string & {}); // not-run: the first four; unavailable: 'no runner'
  readonly runners: { readonly [id: string]: RunnerTests }; // ids assigned by the CLI: display names, de-duplicated 'name', 'name-2', … in candidate order
}

export interface RunnerTests {
  readonly name: string; // display name
  readonly reference?: string; // the reference it was loaded by, only when that isn't a filesystem path
  readonly status: 'completed' | 'error';
  readonly error?: TestError;
  readonly failedRoots?: readonly { readonly dir: RepoPath; readonly error: TestError }[];
  readonly unhandledErrors?: readonly TestError[];
  readonly outOfScope?: number; // listed or run files outside the in-scope tree, counted only
  readonly results: { readonly [file: RepoPath]: readonly TestFileResult[] }; // in-scope files only; one entry per project (TestFileResult.project)
}

/** dev only: pushed through DevContext.onTestUpdate (§5.8). */
export interface TestsUpdate {
  readonly revision: number; // shared with Representation.revision
  readonly status?: Tests['status']; // present when it changed
  readonly reason?: string;
  readonly runners: {
    readonly [id: string]: {
      // only runners with changes; the runner set is fixed per dev session
      readonly status?: 'completed' | 'error';
      readonly error?: TestError;
      readonly results?: { readonly [file: RepoPath]: readonly TestFileResult[] }; // replace per (file, project)
      readonly removed?: readonly { readonly file: RepoPath; readonly project?: string }[];
      readonly failedRoots?: readonly { readonly dir: RepoPath; readonly error: TestError }[]; // replaces when present
      readonly unhandledErrors?: readonly TestError[]; // replaces when present; [] clears
      readonly outOfScope?: number;
    };
  };
}
```

- **Status:** `ran` when at least one runner ran (whatever its outcome); `not-run` with `--no-tests`, when nothing requires tests and no runner exists, during `check`, which lists but never runs (reason `'check'`), and during `init` (reason `'init'`, 02b Q3); `unavailable` when no runner exists while tests are required.
- Results are stored once; nodes reference test files through their `tests` associations. Test files a run reports without having listed them are added to the results and associated weakly (D172).
- **`dev`:** one monotonic revision is shared by data snapshots and test updates; a module applies a message only if its revision is higher than the last one it applied.
- **Merging a `TestsUpdate`** (the CLI and `applyTestUpdate`, C15, merge identically; 02b Fix 3):

  | Field | Rule |
  | ----- | ---- |
  | any field | absent = unchanged; only runners with changes are listed |
  | `results` | each entry replaces the stored entry with the same `(file, project)`; other projects of that file stay; `[]` for a file changes nothing |
  | `removed` | removes entries by `(file, project)`; without `project`, every entry of that file |
  | runner `status` + `error` | present `status` replaces it, and `error` with it (absent `error` = cleared) |
  | `failedRoots`, `unhandledErrors` | present = replaces the whole list (`[]` clears) |
  | `outOfScope` | present = the new total, not a delta |
  | top-level `status` + `reason` | present `status` replaces it, and `reason` with it (absent `reason` = cleared) |
- `TestFileResult` and `TestError` are the runners' raw shapes (§6).

## 5. Rendering module interface (`module/`)

### 5.1 The class

The module's default export is a class (D9). TypeScript can't check static members through `implements`, so `-api` exports the constructor type and authors check against it with `satisfies` (documented in the README sample).

```ts
export interface ShowcaseModuleConstructor<TOptions = unknown> {
  readonly apiVersion: string; // semver range (D13)
  readonly displayName?: string; // for logs and reports; defaults to the reference it was loaded by
  readonly optionsSchema?: JsonSchema; // JSON Schema (draft 2020-12) for moduleOptions: a JSON import or an `as const` literal (D6, Q1)
  readonly rules?: readonly RuleDescriptor[]; // rule ids it can emit, for config validation and docs (§9)
  readonly optionFlags?: { readonly [flag: string]: string }; // D79: CLI flag name → dot path into the options (§5.11)
  readonly kinds?: { readonly [kind: string]: KindDescriptor }; // B9, D162: kinds this module adds
  readonly testRunners?: readonly string[]; // B9, D172: runner references offered as candidates, resolved with the module's file as the parent
  new (init: ModuleInit<TOptions>): ShowcaseModule;
}

export interface KindDescriptor {
  readonly requires: {
    readonly docs?: 'required' | 'optional';
    readonly tests?: 'required' | 'optional';
    readonly showcase?: 'required' | 'optional';
  }; // omitted = optional; unknown keys are ignored (new resource types arrive later as optional arrays on nodes)
}

export interface ModuleInit<TOptions> extends PluginHost {
  readonly options: TOptions; // the root moduleOptions, validated against optionsSchema, defaults applied; per-package values: PackageInput.options (D69, D148)
  readonly repoRoot: string; // absolute
}
```

- **Kinds (D162):** the well-known kinds and their requirements are defined and checked by the CLI (docs for every kind; tests for `component`, `hook`, `function`, `class`; showcases for `component`, `hook`). A module can only **add** kinds; a key that collides with a well-known kind is a load-time error naming the module. Every `FileSymbol.kind` the module emits must be well-known or declared here (§5.4).
- **Test runners (D172):** `testRunners` are references (D9, D114) that join the CLI's candidates before the built-ins; **every reference resolves with the module's file as the parent**: npm names from the module's own location (it depends on them), relative `file:` paths against its file. The module never calls a runner.
- **Rules:** a module declares its rules in its own namespace; declaring one in a CLI-reserved namespace (§9) is a load-time error naming the module.

The CLI constructs **one instance per CLI invocation** and calls its methods in the order of §5.9. Methods may be called concurrently (e.g. `verify` for many files at once), so a module must not keep per-call state on `this`; per-analysis state goes in `AnalysisResult.state`.

### 5.2 Methods: which are required

```ts
export interface ShowcaseModule {
  // Required
  analyze(ctx: AnalyzeContext): MaybePromise<AnalysisResult>;

  // Optional, feature-detected (the CLI reports a clear "module X does not support Y")
  verify?(file: FileInput, ctx: VerifyContext): MaybePromise<readonly Issue[]>;
  resolveRoute?(target: RouteTarget, ctx: RouteContext): string | undefined; // required for build/dev
  build?(ctx: BuildContext): MaybePromise<BuildResult>;
  dev?(ctx: DevContext): MaybePromise<DevSession>;
  scaffold?(ctx: ScaffoldContext): MaybePromise<ScaffoldResult>;
  dispose?(): MaybePromise<void>;
}
```

Rationale (D63, D198): **only `analyze` is required**, because every action starts with it. Every other method is optional, and an action whose method is missing fails with a clear "module X does not support `<action>`" error. Partial modules are legitimate. Action prerequisites are checked when the action starts: `build`/`dev` need `resolveRoute` plus `build`/`dev`; `init` needs `scaffold`. **`verify` is no longer required for `check`** (B9): the CLI performs the representation check itself (D174); a module without `verify` simply adds no framework-specific issues, and `check` says so in one line. `runTests` is gone (B9, D172): the CLI owns the test runners.

### 5.3 Shared context pieces

```ts
/** Lazy, cached, read-only view of the repo (D40). Paths are repo-relative; the boundary is lexical (a path normalising outside the repo root is rejected; symlinks are followed). The CLI records reads for dev invalidation. */
export interface RepoFiles {
  read(path: RepoPath, options?: AccessOptions): Promise<string>; // utf-8; rejects if missing (or ignored, unless options.ignored)
  exists(path: RepoPath, options?: AccessOptions): Promise<boolean>; // false for an ignored path unless options.ignored
  list(dir?: RepoPath, options?: AccessOptions): Promise<DirectoryListing>; // omitted = the root; non-ignored entries unless options.ignored; an ignored dir rejects as missing
}

/** D108: by default an ignored path doesn't exist. `ignored: true` sees ignored paths too, except `.git` and `node_modules`, which are never reachable. */
export interface AccessOptions {
  readonly ignored?: boolean;
}

// One rule for every access method (D108): ignored = nonexistent, unless explicitly asked for.
// Consumers that discover files should not pass `ignored: true` casually (they would find e.g. declarations in `dist/`).
// Plain `fs` would also bypass dev read tracking.
// No `readJson`: manifests are already in `PackageInfo.raw`, tsconfig files are JSONC, and `JSON.parse(await read(p))` is trivial.

export interface DirectoryListing {
  readonly entries: readonly DirectoryEntry[]; // sorted by name (UTF-16 code units)
}

export interface DirectoryEntry {
  readonly name: string; // a name, not a path
  readonly type: 'file' | 'directory'; // of the link target for symlinks; other entry kinds are never listed
  readonly symlink: boolean; // symlinked directories are listed but never walked by the scan
  readonly ignored: boolean; // only ever true with `ignored: true`
  readonly reason?: 'output' | 'skipped' | 'ignore' | 'ignore-file'; // set when ignored: output dir/marker, skipped package, config `ignore`, .gitignore/.tsshowcaseignore
}

/** The repo as scanned by the CLI (C4). `name` is the same value as Representation.home.name. */
export interface RepoInput {
  readonly root: string; // absolute
  readonly name: string;
  readonly info?: PackageInfo;
  readonly vcs?: VcsInfo;
  readonly layout: 'single' | 'workspace';
  readonly packages: readonly PackageInput[];
  readonly files: RepoFiles;
}

export interface PackageInput<TOptions = unknown> {
  readonly dir: RepoPath;
  readonly info: PackageInfo;
  readonly scope: readonly string[]; // B9, D165: the package's effective scope entries (package-relative)
  readonly options: TOptions; // D69, D148: root moduleOptions merged with this package's marked ones, validated
}

// 02b Fix 5a: RepoInput, AnalyzeContext, VerifyContext and ScaffoldContext take the same `TOptions = unknown` parameter and pass it to
// PackageInput; ShowcaseModule<TOptions = unknown> threads it through, so a module typed with its options never casts.

/** B9, D169: the in-scope file tree per package, built by the CLI's scan. */
export interface ScannedTree {
  readonly packages: readonly ScannedPackage[]; // in walk order of their directories
}

export interface ScannedPackage {
  readonly dir: RepoPath; // the package directory ('.' in a single-package repo)
  readonly root: ScannedDirectory; // the package directory, pruned to in-scope directories and the ancestors connecting them
}

export interface ScannedDirectory {
  readonly path: RepoPath;
  readonly inScope: boolean; // false only for ancestors kept to connect in-scope directories to the package directory
  readonly files: readonly ScannedFile[]; // in-scope files only (empty when !inScope), in walk order
  readonly directories: readonly ScannedDirectory[]; // in walk order
}

export interface ScannedFile {
  readonly path: RepoPath;
  readonly name: string;
  /**
   * Which roles are present depends on the pipeline stage (B9, D169, G6):
   * - in AnalyzeContext.tree, only 'doc' and 'config' (the CLI's scan-time roles): 'exporting' and 'showcase' are what the
   *   analysis itself produces, and 'test' is left out because the CLI lists test files in parallel with the analysis;
   *   a module that needs to know about tests matches file names itself;
   * - in every later context (verify, scaffold, build, dev), every role.
   */
  readonly roles: readonly FileRole[];
}

export type FileRole = 'exporting' | 'doc' | 'test' | 'showcase' | 'config' | (string & {}); // open union
```

- **Walk order** (everywhere in the contract): a directory's files before its subdirectories, names sorted in UTF-16 code units, recursively; identical on every machine.
- **Roles:** `doc` = a `.md` file, or a non-markdown text form only where it is accepted: `README`/`README.txt` in any directory, `LICEN[CS]E`/`CHANGELOG` (and `LICEN[CS]E.txt`) only in a home's root (§4.3); so e.g. `src/vendor/LICENSE` has no role; `config` = a directory config file (D188); `test` = listed by a runner and in scope (D172); `exporting` = at least one symbol; `showcase` = at least one showcasing component. A file may have several roles; `test` and `showcase` are exclusive (a file that is both counts as a test).
- **In scope** (D165): at or below a match of the package's `scope`, below the package directory, not below a nested package, not ignored or skipped. Directories outside every package are never in scope.

### 5.4 The analysis (D5, D170, D171)

Runs once per action, and again in `dev` when the structure changes or any of its `inputs` changes. Typically uses the TS compiler API against the real filesystem (`repo.root`), reading entries and barrels wherever they are, so its reads are not tracked through `RepoFiles`; instead it **reports the files it depended on** (`inputs`, D68).

```ts
export interface AnalyzeContext {
  readonly repo: RepoInput;
  /** Roles here are only 'doc' and 'config' (see ScannedFile.roles): exporting/showcase come from this analysis; tests are listed in parallel. */
  readonly tree: ScannedTree;
  readonly signal: AbortSignal; // dev aborts a stale analysis
}

export interface AnalysisResult {
  readonly files: {
    readonly [path: RepoPath]: {
      readonly symbols?: readonly FileSymbol[]; // in declaration order
      readonly showcasingComponents?: readonly ShowcasingComponent[];
      readonly meta?: JsonObject; // module data for this file, handed back in FileInput.meta
    };
  };
  readonly issues?: readonly Issue[]; // package- or repo-level, or about one file (placement: §9)
  readonly inputs?: readonly RepoPath[]; // files the analysis depended on; dev re-analyses when any changes (D68)
  readonly state?: unknown; // opaque, handed back to verify, resolveRoute, scaffold, build and dev; never serialised
  readonly meta?: JsonObject; // becomes Representation.meta
}

export interface FileSymbol {
  readonly name: string; // the declared name ('default' when the declaration has no identifier)
  readonly kind: SymbolKind; // well-known or declared in the module's `kinds`
  readonly meta?: JsonObject; // copied verbatim to RepresentedItem.meta
}

export interface ShowcasingComponent {
  readonly name: string; // identity: the export name ('default' for the default export); unique per file
  readonly meta?: JsonObject; // copied verbatim to ShowcaseAssociation.meta (e.g. the shell opt-in, a title)
}
```

- **Attachment (D171):** a symbol attaches to the file that **declares** it (its value declaration's file; type-only symbols to their first declaration outside a module augmentation); being attached means it is publicly exported. One symbol per (file, name); with declaration merging the value meaning decides the kind, `type` only without one; namespaces holding only types are `type`. A JSON module's default export attaches to the `.json` file. Declarations outside the package, in `node_modules` or in ignored files aren't attached; the module may report them through `issues`. In-source tests (Vitest `includeSource`) are both test files and exporting files, and associate with themselves. Exported alias names are deferred.
- **Showcases (D195):** several showcasing components per file, identity = export name; a showcase file counts only if at least one was found.
- **Validation by the CLI (D170, replaces D70's unit checks):** `files` keys outside scope are dropped silently; a key naming a nonexistent file, or two keys normalising to the same path, is a thrown error naming the module. A kind that is neither well-known nor declared gives a `module/error` issue on that file, and the symbol is treated as `constant`. Kind overrides from configuration (D189) are applied by the CLI afterwards; every later context sees effective kinds.
- **Packages are always present in the data (D193):** the analysis doesn't choose packages; a module that wants to hide one does so when presenting.

### 5.5 `verify` (per file, D177)

```ts
export interface FileInput {
  readonly path: RepoPath;
  readonly package: RepoPath;
  readonly roles: readonly FileRole[]; // every role (§5.3)
  readonly symbols: readonly RepresentedItem[]; // effective kinds (after overrides)
  readonly showcasingComponents: readonly ShowcasingComponent[];
  readonly meta?: JsonObject; // AnalysisResult.files[path].meta
  readonly associations: readonly {
    readonly node: string; // node id
    readonly level: AssociationLevel;
    readonly type: 'doc' | 'test' | 'showcase' | (string & {});
  }[]; // this file as a resource
  readonly nodes: readonly string[]; // the file's own nodes, else [the nearest node above]: where its issues attach
  read(): Promise<string>; // lazy, cached, tracked (D40)
}

export interface VerifyContext {
  readonly repo: RepoInput;
  readonly tree: ScannedTree; // every role
  readonly representation: Representation; // tests.status 'not-run', reason 'check'
  readonly package: PackageInput; // the file's package, with its options
  readonly state: unknown; // AnalysisResult.state
}
```

- Called for **every in-scope file**, after the representation is built (§5.9); possibly concurrently. There is no `ignored` verdict: a file with nothing to say returns `[]`.
- Returned issues get the input file as their `file` (the CLI forces it) and attach to `FileInput.nodes`. Issues about other files belong to that file's own `verify` call or to the analysis.
- **A throwing `verify`** doesn't abort `check`: the CLI turns it into a `module/error` issue on that file (severity `error`, naming the module) and continues.
- **`dev` invalidation:** a file whose tracked reads changed is re-verified; any change to enrichments, associations, configs or the representation re-verifies every file.

### 5.6 `resolveRoute` and links (D67, D183)

```ts
export interface RouteTarget {
  readonly node: string; // node id
  readonly doc?: RepoPath; // set iff the link target is a doc in the selected node's docs
  readonly anchor?: string; // only with doc: the rendered, doc-id-prefixed element id
}

export interface RouteContext {
  readonly representation: Representation; // the representation the route belongs to (in dev, the current one)
  readonly state: unknown; // AnalysisResult.state
}

// resolveRoute(target, ctx) → an in-app URL relative to the app's index.html (e.g. '#/button', '#/button?h=doc-3f2a9c1e--props'), or undefined
```

- **The CLI decides which node a link targets; the module only turns a target into a URL.** The CLI calls `resolveRoute` for every node (storing `route`, D67) and for every in-app link and anchor in rendered docs (C5). The browser app reads routes from the data: one source of truth.
- **`undefined`** means "no in-app page": the node's `route` stays unset and links to it go to the VCS host. A module may answer for a node it doesn't present with another node's URL (e.g. a covering node).
- The returned URL must work under any base path: relative, no leading `/`. **Purity:** the result depends only on its arguments. Synchronous (principle 8).
- **Link resolution (static, independent of routes; D124 holds):** (0) a directory with a `README` stands for that README; (1) nonexistent or ignored → broken; (2) a home doc, or a repo or package root directory → its home; (3) out of scope or outside every package → the VCS host; (4) candidates: for a file its own nodes (its direct node, its collection's node, configured nodes listing it, except upward listings; counting as primary), else the nodes it is associated with; for a directory without a README its own node only; selected by strongest association, then shallowest tree depth, then tree order; no candidates: a file → the VCS host (D73), a directory → broken.
- **Rendering of links (C5):** broken links are an `<a>` without `href` with a catalogue state attribute, plus `docs/broken-link`; VCS links use `VcsInfo.fileUrlTemplate`, fragments passed through; with no VCS info they are an `<a>` without `href` and no issue (one `build` warning, D110). Fragments on doc targets are rewritten to the prefixed id and passed as `anchor`; on other in-app targets they are dropped; a fragment missing from the target doc gives `docs/broken-anchor`. A doc's own heading permalinks point at its **link-selection node**. External links are untouched; local media become doc assets (D120).

### 5.7 Tests (removed by B9)

`runTests`, `TestContext`, `loadTestRunner`, `LoadTestRunnerOptions`, `LoadedTestRunner`, `TestResults` and `TestResultsUpdate` are gone (D172). The CLI selects runners (candidates: the module's static `testRunners`, the built-ins, config `testRunners`; config `testRunner` pins), lists and runs tests, intersects the results with the in-scope tree and stores them in `Representation.tests` (§4.4); in `dev` it streams `TestsUpdate`s through `DevContext.onTestUpdate`. The runner contract itself is §6. `TestFileResult.file` is always repo-relative.

**Runner outcomes (D172; the CLI owns them, C6 and C11 implement them):**
- a `detect` that throws → `tests/runner-failed`, that runner unused; a pinned runner skips `detect`;
- a listing with `status: 'error'` (or a rejection) → `tests/runner-failed`, and every `represent/missing-tests` is suppressed; a listing's `failedRoots` suppress it for symbols declared under those directories;
- a run that fails (`status: 'error'`, a rejection) → `tests/runner-failed`; the other runners' results are still used;
- no runner while at least one symbol requires tests → `tests/no-runner`, every `represent/missing-tests` suppressed, `tests.status: 'unavailable'`;
- no symbol requires tests (after kind overrides, module kinds, exemptions) → short-circuited: no `tests/no-runner`, `tests.status: 'not-run'`, reason `'no tests required'`; detected runners still list and run existing tests;
- a file listed by several runners runs with each; in `dev`, listing is redone on structure changes (e.g. a new test file), so new files get the `test` role.

### 5.8 `build` and `dev` (D43, D187)

```ts
export interface BuildContext {
  readonly representation: Representation; // B9: replaces data: ShowcaseData
  readonly docs: DocsAccess; // B9: lazy doc rendering for this representation
  readonly state: unknown; // AnalysisResult.state
  readonly themes: readonly Pick<ThemeDefinition, 'name' | 'label' | 'scheme'>[]; // in config order; safe to embed as is (D102)
  readonly themeStylesheets: ThemeStylesheetFiles; // B1, D82: absolute paths to the CLI-scoped stylesheets (§8 rule 3). Toolchain inputs: CSS with url()s relative to the file, pointing at source assets; the module emits those assets and rewrites the URLs (or passes a matching baseURL to constructed sheets) (03b H8, 03b H35)
  readonly defaultTheme?: string; // explicit config choice; otherwise the inclusion rules decide (§8)
  readonly repoRoot: string; // absolute; to turn RepoPaths (showcase files) into imports
  readonly cacheDir: string; // absolute, module-private scratch space, persists across runs
  readonly signal: AbortSignal;
}

export interface DocsAccess {
  render(ref: DocRef): Promise<RenderedDoc>; // cached; bound to the representation this DocsAccess came with; rejects (naming the doc) for a ref not in it
  read(ref: DocRef): Promise<string>; // the raw source; same rejection
}

export interface BuildResult {
  readonly files: readonly BuildFile[]; // the site, except doc assets, which the CLI adds (B6, D120); the CLI writes it (D64)
}

export interface BuildFile {
  readonly path: string; // relative to the site root, POSIX; no '..', no absolute paths (the CLI rejects them)
  readonly contents: string | Uint8Array;
}

export interface DevContext extends BuildContext {
  readonly host?: string;
  readonly port?: number;
  onDataUpdate(listener: (update: DataUpdate) => void): Unsubscribe; // a full snapshot each time (Q6)
  onTestUpdate(listener: (update: TestsUpdate) => void): Unsubscribe; // B9, D186
  onThemeStylesheetsUpdate(listener: (sheets: ThemeStylesheetContents) => void): Unsubscribe; // 03b H7: new stylesheet CONTENTS after a theme file change (the files are also rewritten in place)
  readonly docAssetsDir: string; // B6, D125: absolute; a CLI-maintained directory laid out like the site's tss-assets/, holding, in dev, the assets of **every** DocRef (pre-copied before each onDataUpdate, 02b Q2); the module serves it statically at tss-assets/ and nothing else under that prefix
}

/**
 * The part of the context that changes during a dev session (02b Q1). Every other DevContext member is fixed for the session:
 * a config change restarts dev; theme stylesheet contents and test results have their own channels.
 */
export interface DataUpdate {
  readonly representation: Representation;
  readonly docs: DocsAccess; // bound to this representation; the previous DocsAccess must not be used for its refs
  readonly state: unknown; // the AnalysisResult.state this representation was built from
}

export interface ThemeStylesheetFiles {
  readonly shell: string; // absolute path of the CLI-scoped shell stylesheet
  readonly fonts: string; // absolute path of the fonts stylesheet
}

export interface ThemeStylesheetContents {
  readonly shell: string; // the new CSS contents (the files at ThemeStylesheetFiles are rewritten in place too)
  readonly fonts: string;
}

export interface DevSession {
  readonly url: string;
  readonly closed: Promise<void>; // settles when the server stops; rejects if it fails after starting
  close(): Promise<void>;
}
```

- **The module returns the site; the CLI writes it (D64).** `BuildContext` has no `outDir`. The CLI validates the returned paths, checks the inclusion rules it can check (e.g. `index.html` exists, no absolute build-machine paths, §8 rules 1–2), adds the doc assets and writes `outDir` with the output guard, an atomic swap and its marker file (C11).
- **Lazy docs (D187, E2):** the module renders the docs it presents through `docs.render`; **only rendered docs and their assets reach the `build` output**: a doc's assets are written iff `render` was called for it in this build. In `dev` (nothing is published), the CLI pre-copies the assets of every `DocRef` into `docAssetsDir` before each `onDataUpdate`, so the folder is complete when the update arrives (02b Q2). In `dev`, any change that affects a rendered doc, a content-only edit included, produces a new snapshot through `onDataUpdate`, and the module renders again from that snapshot's `DocsAccess` (no separate docs signal for now; one can be added later, 02b H2). How rendered HTML is embedded in the bundle is a module/`-base-vite` mechanic (C15).
- **Toolchains:** Vite builds in memory (`build.write: false`), which `-base-vite` maps to `BuildFile[]` (C15). A toolchain that can only write to disk builds into `ctx.cacheDir` and returns what it wrote.
- `dev` writes nothing: dev servers serve from memory.
- In `dev`, test updates reach the dev server separately (`onTestUpdate`) so a test run doesn't force a full data push; the CLI merges them into its own copy, so the next `onDataUpdate` snapshot is consistent (one shared revision, D186).
- **Shutdown order** (Ctrl-C or a fatal error): the CLI aborts every `signal` (stopping analysis, listing and test watching) → `DevSession.close()` → `dispose()`. A rejected `DevSession.closed` ends `dev` with the error.
- Theme CSS edits in `dev`: the CLI regenerates both `themeStylesheets` in place (D82) and pushes the new contents through `onThemeStylesheetsUpdate` (03b H7, D90); the module re-reads the files or applies the pushed contents (`-base-vite` re-reads the files and hot-reloads them, C15, B8). Changing which themes are configured needs a restart.

### 5.9 Call order per action (D168)

| Action  | Calls |
| ------- | ----- |
| `check` | scan → CLI enrichments (docs, configs, test listing) in parallel with `analyze` → kind overrides → associations → representation → the CLI's checks (representation; links and anchors of every doc, home docs included; in no particular order) → `verify` × every in-scope file (if the module has it) → issues attached, severity overrides applied → report |
| `init`  | as `check` up to the representation → `scaffold` → the CLI writes the returned files |
| `build` | the full `check` (report printed; it fails `build` only per `--strict`, D164) → tests run by the CLI (unless `--no-tests`, D197) → `resolveRoute` for every node → `build` (rendering docs through `docs`) → the CLI writes the output, then applies `--strict` |
| `dev`   | as `build` (issues shown, never fatal), with tests in watch mode and `dev`; on file changes the CLI rescans / re-analyses / re-verifies as needed and pushes `onDataUpdate` (also for content-only doc edits) and `onTestUpdate` |
| all     | `dispose` at the end, also on Ctrl-C |

### 5.10 `scaffold` (D16, D196)

The module plans the files; the **CLI writes them** (one implementation of dry-run, never-overwrite-silently and reporting). The `init` semantics are C8's (D196).

```ts
/** Built up to the representation (02b Q3): node `issues` are empty and tests have not run (tests.reason 'init'). Decide what is missing from
 *  each node's docs, tests and showcases and the kinds' requirements, not from issues. */
export interface ScaffoldContext {
  readonly repo: RepoInput;
  readonly tree: ScannedTree; // every role
  readonly analysis: AnalysisResult;
  readonly representation: Representation; // B9: init runs the pipeline up to the representation
  readonly state: unknown; // AnalysisResult.state
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
// Real modules keep the schema in options.schema.json with a sidecar (D62); this sample inlines it so it type-checks on its own (02b Fix 7a).
const optionsSchema = { type: 'object', properties: { homeDocs: { type: 'array', items: { type: 'string' } } }, additionalProperties: false } as const;
type Options = OptionsOf<typeof optionsSchema>; // via tsschema: { homeDocs?: string[] }
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
static readonly optionFlags = { 'home-docs': 'homeDocs', kind: 'scaffold.kind' } as const;
```

- **Precedence:** root config < package config < CLI flags (D6). The CLI applies flags on top of the root options and every merged per-package result, then validates, so every method simply sees options (`ModuleInit.options`, `PackageInput.options`).
- **Parsing from the schema** at the target path: booleans become `--flag` / `--no-flag`, numbers are coerced, arrays come from repeating the flag, enums are validated by ajv.
- **`--help`** lists the module's flags with the target property's schema `description`.
- **Collisions:** a flag that clashes with a CLI flag is a load-time error naming the module; the CLI reserves its own names.
- Flags are accepted on every action, even where unused. Options that only make sense as flags (e.g. `scaffold.kind`) are still valid in config files; their schema `description` says so.

`testRunner` is **not** a module option any more (B9, D172): `testRunner` and `testRunners` are CLI config keys (§10), and a module offers runners through its static `testRunners` (§5.1).

**Path-valued options (B5, D116).** A module marks string options holding filesystem paths with `format: 'tsshowcase-path'` and options holding plugin references with `format: 'tsshowcase-reference'` (`OPTION_FORMAT = { path, reference } as const`, exported from `module/`). Before merging root config, package configs and flags, the CLI makes relative values absolute against the base of the source they were written in (the config file's directory, the package config's directory, the process cwd for flags; D112); for references only `file:` values are rewritten (D114). Markers may sit on properties, array `items` and `additionalProperties` (through local `$ref`s), never under `oneOf`/`anyOf`/`allOf`/conditionals (a load-time error). Modules therefore see absolute paths and absolute `file:` references only. Mechanics: C6 §9.4.

**Per-package options (B8, D148).** Module options are **root-only by default**: one showcase build and one document mean most options exist once per repo. A module opts a top-level property of its options schema into per-package values with the custom keyword `"x-tsshowcase-per-package": true` (`PER_PACKAGE_OPTION_KEYWORD`, exported from `module/` next to `OPTION_FORMAT`). It sits beside the property's ordinary keywords, so `format` keeps its meaning (`{ "type": "array", "items": { "type": "string" }, "x-tsshowcase-per-package": true }`). The CLI registers the keyword with ajv as an annotation. In a package config's `moduleOptions`, only marked keys are merged (D69); any other key is warned about ("`<key>` can only be set in the root config") and dropped before merging (C6 §7). Only top-level properties count: per-package options merge shallowly. A module that forgets to mark an option therefore fails loudly (the warning) rather than silently ignoring a package value.

## 6. Test-runner interface (`testrunner/`)

```ts
export interface TestRunnerConstructor {
  readonly apiVersion: string;
  readonly displayName?: string;
  new (host: PluginHost): TestRunner;
}

export interface TestRunner {
  detect(repo: RunnerRepoInput): MaybePromise<boolean>; // auto-detection; skipped when the runner is pinned (D172)
  listTestFiles(options: TestListOptions): Promise<TestListing>; // B9, D172: required; used by every action, check included
  run(options: TestRunOptions): Promise<RawTestReport>;
  watch(options: TestRunOptions, onUpdate: (update: RawTestUpdate) => void): Promise<RawTestReport>; // resolves with the first full run; stops on options.signal
}

export interface RunnerRepoInput {
  readonly root: string; // absolute
  readonly layout: 'single' | 'workspace'; // B7
  readonly info?: PackageInfo; // B7: the root manifest, if any; the CLI passes its RepoInput (assignable)
  readonly packages: readonly RunnerPackageInput[]; // 02b Fix 1e: no module options; PackageInput stays assignable
  readonly files: RepoFiles;
}

export interface RunnerPackageInput {
  readonly dir: RepoPath;
  readonly info: PackageInfo;
}

export interface TestListOptions {
  readonly repo: RunnerRepoInput;
  readonly signal: AbortSignal; // 02b Fix 1d: an aborted listing resolves with status 'error', like aborted runs
}

export interface TestRunOptions {
  readonly repo: RunnerRepoInput; // B7: replaces root/repoRoot; the runner decides where its tool runs (D132) and reports paths relative to repo.root
  readonly files?: readonly RepoPath[]; // B7: optional filter, repo-relative files or directories (prefix match); the CLI passes none in v1: runs are unfiltered, then intersected with the in-scope tree (D172)
  readonly signal: AbortSignal;
}

/** B9, D172: what a runner would run, without running it. */
export interface TestListing {
  readonly status: 'completed' | 'error'; // 'error' = listing failed entirely (never throws where it can, like run)
  readonly error?: TestError;
  readonly files: readonly { readonly file: RepoPath; readonly project?: string }[]; // repo-relative; identity (file, project)
  readonly failedRoots?: readonly { readonly dir: RepoPath; readonly error: TestError }[]; // directories where listing failed while others worked
}

export interface RawTestReport {
  readonly status: 'completed' | 'error'; // 'error' = the runner itself failed (stored per runner in Representation.tests; --strict=unavailable)
  readonly error?: TestError;
  readonly files: readonly TestFileResult[]; // "no tests found" is status 'completed' with files: [] (B2)
  readonly unhandledErrors?: readonly TestError[]; // errors not attached to a file, e.g. TS syntax errors that make type tests unreliable (B2)
  readonly failedRoots?: readonly { readonly dir: RepoPath; readonly error: TestError }[]; // B7, D135: directories where the tool couldn't run while others did (all of them with status 'error'); stored per runner, --strict=unavailable (B9)
}

export interface RawTestUpdate {
  readonly files: readonly TestFileResult[]; // full replacement per listed (file, project)
  readonly removed?: readonly { readonly file: RepoPath; readonly project?: string }[]; // best-effort; the CLI's dev watcher is the reliable source of deletions (B2)
  readonly unhandledErrors?: readonly TestError[]; // B7: when present, replaces the previous set (an empty array clears it); absent leaves it unchanged
}

export interface TestFileResult {
  readonly file: RepoPath;
  readonly project?: string; // B2: the test tool's project name; a file can appear once per project, so identity is (file, project)
  readonly status: TestStatus;
  readonly durationMs?: number;
  readonly error?: TestError; // file-level failure: a syntax error (then without children) or e.g. an afterAll failure (B7)
  readonly children: readonly TestNode[];
}

export type TestNode =
  | { readonly type: 'suite'; readonly name: string; readonly status: TestStatus; readonly error?: TestError; readonly children: readonly TestNode[] } // error: e.g. a failing beforeAll (B7)
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
- **Leaves the process as it found it (B2, refined by B7, B9):** around `listTestFiles` and in `run`, a runner restores `process.env` and `process.exitCode` after the run; in `watch`, `process.exitCode` after every rerun (only when the tool changed it) and `process.env` when watching stops (Vitest sets `NODE_ENV=development` and `exitCode = 1` on failures, which would turn a later `vite build` into a dev build and make `build` exit non-zero, against D8). A runner also removes process-wide signal, exit and `unhandledRejection` handlers its tool installs (Vitest's call `process.exit()`).
- **Aborts don't throw (B7):** an aborted `run`/`watch` resolves with `status: 'error'`.
- **No tests found** is `status: 'completed'` with no files, not an error (B2).
- Type tests (`*.test-d.ts`, D38) appear as ordinary files and tests.
- **Listing (B9, D172):** `listTestFiles` returns the files the runner would run, using the same run plan as `run` (D132–D134); the CLI intersects them with the in-scope tree to give files the `test` role. A listing with `status: 'error'` (or a rejection) gives `tests/runner-failed` and suppresses every `represent/missing-tests`; `failedRoots` suppress it for symbols declared under those directories.
- The runner knows nothing about symbols or nodes; the **CLI** maps results through the test-file associations (B9, D172).

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
  readonly scheme: 'light' | 'dark'; // drives the prefers-color-scheme default and `color-scheme` (D7, D82, D89)
  readonly css: readonly string[]; // absolute paths to the theme's CSS files, in cascade order; unscoped (see below)
}

export interface ResolvedTheme extends ThemeDefinition {
  readonly reference: string; // the config reference it came from
}
```

**Reference syntax:** `<ref>` selects every theme in the set; `<ref>#<name>` selects one. Built-in names (`default-light`, `default-dark`) are aliases the CLI maps to `@ofzza/tsshowcase-theme-default#default-light` etc.

**Names and assets.** Two configured themes with the same `name` are a CLI error. `url()` references in theme CSS (fonts, images) resolve relative to their CSS file, and the build must bundle them into the output (§8 rule 2 bans third-party origins).

**Theme CSS is unscoped; the CLI scopes it (D65, D82).** A theme's CSS files must not reference the theme's own name or `THEME_ATTRIBUTE`. The CLI turns all configured themes into a **shell stylesheet** (every theme wrapped in `:where(.tss-shell[data-theme="<name>"])`) and a **fonts stylesheet** (`@font-face` rules), handed over as `BuildContext.themeStylesheets` (mechanics: C3 plan §5).

**What a theme may do (B1, D86, D89):** a theme only ever applies inside the shell's shadow root (§8 rule 7), so it may style any catalogue class, bare elements **inside `.tss-content` only** (D92), and add its own variables, classes and selectors, within four rules (D86): (1) no self-scoping, no document selectors, nothing reaching showcases (`[data-theme]`, `:host`, `:host()`, `html`, `body`, `slot`, `::slotted()`, `::part()`, compound `:root`, `color-scheme`); (2) its own names (classes, variables, keyframes, font families) use its own prefix, never `tss-`/`--tss-`; (3) modules only use catalogue names in markup (a catalogue class on every element a theme may style), plus their own prefixed arrangement-only classes, which themes never style; (4) respect the shell's layout rules (C3 §1). Documented with their purpose in "Building a theme that works with any module" (C3 §11). `@ofzza/tsshowcase-lint` offers presets that check them (D88). A composing theme instantiates the base `ThemeSet` directly, without an `apiVersion` check; that's acceptable because npm resolution pins the base package to the range the composing theme declares (K28).

**Extending a theme is composition, not a contract feature (D65).** There is no `extends` field. A theme package that builds on another depends on it as an ordinary npm dependency (so npm resolution, pinned by its own range, picks the actual package) and puts the base theme's CSS files before its own overrides:

```ts
// my-ocean-theme/src/index.ts  (package.json: "dependencies": { "@ofzza/tsshowcase-theme-default": "^0.1" })
import { fileURLToPath } from 'node:url';
import DefaultThemes from '@ofzza/tsshowcase-theme-default';
import type { PluginHost, ThemeSet, ThemeDefinition, ThemeSetConstructor } from '@ofzza/tsshowcase-api';

export default class OceanThemes implements ThemeSet {
  static readonly apiVersion = '^0.1';
  readonly themes: readonly ThemeDefinition[];

  constructor(host: PluginHost) {
    const base = new DefaultThemes(host).themes.find((theme) => theme.name === 'default-light')!;
    this.themes = [{ name: 'ocean', label: 'Ocean', scheme: 'light', css: [...base.css, fileURLToPath(new URL('../css/ocean.css', import.meta.url))] }];
  }
}
OceanThemes satisfies ThemeSetConstructor; // checks the static side
```

_This sample is a `nocheck` fence in the `theme/` README until C3 implements `-theme-default`; C3 turns the check on (02b Fix 7a)._

`css/ocean.css` lives outside `src/` (a pure `tsc` build doesn't copy CSS) and is published with `files: ["dist", "css"]`, as `-theme-default` does (G33). It overrides only what it needs (e.g. the primary/secondary base tokens), and the variable hierarchy cascades it (C3). Config: `themes: ['my-ocean-theme']`, or mixed with built-ins. This only works because of the scoping rule above: self-scoped base files would keep targeting `default-light`. The `theme/` README documents the pattern (the sample is type-checked, D56). A CLI-resolved `extends` reference could still be added later, additively (D13), if composition proves clumsy.

## 8. Output inclusion rules (`inclusion/`, D43)

Binding on **every** module's output, whether or not it uses `-base-vite`. Documented in the `inclusion/` README; constants exported so nobody hardcodes the strings.

**Not a rule: how the data is embedded (D77).** How the `Representation` and the rendered docs get into the site (compiled into the bundle, a fetched `data.json`, inlined in `index.html`, …) is the module's business: nothing outside the module's own app reads it. Rules 1 and 2 (static and relocatable, no leaked absolute paths) are the only constraints, plus one from B9 (D187): **only rendered docs and their assets reach the output**. `-base-vite` compiles the data into the bundle as a virtual module; how rendered docs travel is designed in C15's redesign amendment.

0. **The CLI alone writes the output (D64).** `build` returns the site as `BuildFile[]`; a module that writes the output directory itself breaks the contract.
1. **Static and relocatable.** The returned files, plus the doc assets the CLI adds, form a self-contained static site with `index.html` at its root. The prefix `tss-assets/` (`DOC_ASSETS_DIR`) is reserved for doc assets, written by the CLI; a module returns no file under it (B6, D120). It works when served from any sub-path (GitHub Pages project sites): all URLs relative, routing needs no server rewrites (hash routing or equivalent).
2. **No leaks.** No absolute filesystem paths from the build machine appear in the output. No runtime requests to third-party origins (fonts, CDNs, analytics) unless the user's own showcase code or README content (external images, badges) makes them (B6).
3. **Every configured theme is included** (B1, D82, D89; **binding**, 03b H34): the shell lives in a **shadow root** on a host element `SHELL_HOST_TAG` (`'tsshowcase-app'`); the module adopts the CLI's **shell stylesheet** (`BuildContext.themeStylesheets.shell`) into it **first**, then its own CSS (`adoptedStyleSheets = [themeShell, ...moduleSheets]`; a `<style>` fallback keeps the same order, never mixed), and adds the **fonts stylesheet** (`.fonts`) to the document. The module processes both files as toolchain inputs (emitting their assets, 03b H8) and, in `dev`, applies the updated stylesheets after `onThemeStylesheetsUpdate` (B8). Inclusion constants (`inclusion/`): `SHELL_HOST_TAG`, `THEME_ATTRIBUTE`, `THEME_STORAGE_KEY`, `SHOWCASE_WRAPPER_ATTRIBUTE` (`'data-tss-showcase'`), `SHOWCASE_SLOT_PREFIX` (`'tss-showcase-'`) (K16), `DOC_ASSETS_DIR` (`'tss-assets'`, B6). A theme is applied by setting `THEME_ATTRIBUTE` (`'data-theme'`) to its `name` on the shell root element (`tss-shell`) inside the shadow root; shell overlays render inside that element (03b H29). Theme CSS never applies to showcase content. Theme rules carry no scoping specificity (`:where()`), so the module's later CSS wins by order where both set a property. No cascade `@layer` (D81).
4. **Initial theme selection,** applied before the shell first paints (no flash): the stored choice (`localStorage[THEME_STORAGE_KEY]`, `'tsshowcase:theme'`) if it names an included theme → else the configured `defaultTheme` → else the first theme whose `scheme` matches `prefers-color-scheme` (read once at start; later OS changes are ignored in v1, 03b H16) → else the first theme. With only one theme, no switcher is rendered. **The page itself (03b H6):** the module sets `color-scheme` and the canvas background on `<html>` from the active theme (kept in sync on switch), and `index.html` ships `<meta name="color-scheme" content="light dark">` plus a tiny inline script applying this selection to `<html>` before the bundle loads; this is the only document-level styling besides the fonts stylesheet and the user's showcase stylesheets.
5. **The switcher** stores the user's choice under `THEME_STORAGE_KEY` and uses the catalogue's switcher class (C3).
6. **Doc HTML (`RenderedDoc.html`) is inserted verbatim** inside an element with the catalogue class `tss-content`, inside the shell (D89). It is themed like any native part of the shell: C5's HTML carries catalogue classes where an element needs one, and the theme styles the rest; links and element ids stay intact. Element ids are prefixed with the doc id (B9, D184), so several docs can share a page; the module scrolls to them itself (fragment navigation doesn't reach shadow trees). URLs in doc HTML (doc assets) are relative to the site root; a module inserting doc HTML into a document elsewhere sets `<base>` or rewrites them (B6).
7. **Showcases are isolated from the shell (B1, D89; how is **free**, 03b H34).** A showcase is never styled by the shell, and the shell's elements are never styled by a showcase's CSS, while the showcase's portals and overlays (modals, dropdowns, tooltips) keep working. Each showcase is rendered into a light-DOM wrapper `[data-tss-showcase]` that plays the role `body` plays in the user's app, and modules offer a **showcase stylesheet** option (document-level CSS where users put their app typography, e.g. `body, [data-tss-showcase] { … }`); with it a showcase looks as in the user's app (03b H1). Every showcase receives the selected colour scheme (`'light' | 'dark'`) through a **module-defined showcase interface** (for React, e.g. a `colorScheme` prop); the module pairs each showcase with its own light/dark selector (following the theme's `scheme` until changed manually; a theme switch resets it; not persisted, 03b H12) and gives the stage a backdrop matching it (`data-scheme`). How isolation is achieved is the module's business; the **recommended technique** (used by `react-default`, specified in C3 §1): showcases are light-DOM wrappers assigned to named slots in their stage (`:host { all: initial !important; display: block !important; isolation: isolate !important }`; **every declaration after `all: initial !important` must carry `!important`**, or it silently loses (B2, verified), `slot { all: initial !important; display: contents !important; color-scheme: <selected> !important }`), with document scrolling and no clipping or containing-block properties on stage ancestors, and the host as a stacking context. Modules also offer a **showcase setup** option: a document-level module run before showcases mount (Storybook-preview style), e.g. for libraries that manage document-wide focus and must be told about Shadow DOM (Fluent v9's Tabster: `window.__tabsterShadowDOMAPI = getShadowDOMAPI()`) (B2). Slotted content stacks inside the host's stacking context, so a showcase's fixed/absolute element without a high z-index sits under the shell's z-indexed elements; body portals paint above the shell. React synthetic events from slotted showcases bubble to the shell's React ancestors, so the shell attaches no handlers on stage ancestors that react to them. A showcase may opt, as a whole, into rendering **inside** the shell to get the theme (only for components styled purely by catalogue classes and variables; needed for showcases of shell components, D41); such a showcase follows the shell's theme and has no scheme selector (D100). The opt-in is per showcasing component, carried in `ShowcaseAssociation.meta` under a module-defined key (B9, D195; React: `shell`, C8).

## 9. Issue model (`issues/`)

```ts
export type Severity = 'error' | 'warning';

export interface Issue {
  readonly rule: string; // namespaced id, e.g. 'react/showcase-format', 'headless/render-pure'
  readonly severity: Severity; // the emitter's default; the CLI applies overrides (below)
  readonly message: string; // one sentence, no trailing period
  readonly package?: RepoPath; // derived by the CLI from `file` when unset (D70)
  readonly file?: RepoPath; // always repo-relative; set by the CLI for verify issues (the input file)
  readonly line?: number; // 1-based
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

- **B9:** `Issue.dir` is gone (no directory verdicts). **Each issue lives in exactly one place** (D180, 02b Fix 4b), decided by the CLI:

  | Issue | Attached to |
  | ----- | ----------- |
  | `represent/*` | every source considered: the symbol's direct or collection node and each configured node listing it |
  | `docs/*` | the nodes holding the linking doc's `DocRef` (a home too, for a broken link in a home doc) |
  | `config/*` | the config directory's own node, else the nearest node above; root-config issues → `Representation.issues` |
  | `verify` issues, analysis issues with an in-scope `file` | the file's own nodes, else the nearest node above (`FileInput.nodes`) |
  | `scan/invalid-manifest`, other package-level issues | that package's root |
  | `scan/unreadable` | the nearest node above the directory; outside every package → `Representation.issues` |
  | `tests/no-runner`, `tests/runner-failed`, `scan/no-manifest`, other repo-level issues | `Representation.issues` only |

  "The nearest node above" is always a root or indirect node, never a home; homes carry only issues about their own docs. The module decides how to show `Representation.issues` (e.g. a banner on the home page).
- **Rule ids (D175).** The CLI's own ids and default severities: `represent/missing-docs`, `represent/missing-tests`, `represent/missing-showcase`, `tests/no-runner`, `tests/runner-failed`, `config/several-configs`, `config/collection-clash`, `config/collection-membership`, `config/kind-conflict`, `config/same-base`, `module/error`, `scan/invalid-manifest` (errors); `config/unused-key`, `config/missing-path`, `config/no-role`, `config/out-of-scope`, `config/shadowed`, `config/slug`, `docs/broken-link`, `docs/broken-anchor`, `scan/no-manifest`, `scan/unreadable` (warnings). The namespaces `represent/`, `tests/`, `config/`, `docs/`, `scan/` and `module/` are **reserved for the CLI** (02b H4); a module's rules use its own namespace (e.g. `react/`, `headless/`) and are declared in its `rules`. A module declaring a rule in a reserved namespace fails to load; an issue it emits with such a rule becomes a `module/error` (C6).
- **Severity overrides are applied by the CLI, not the emitter (D176).** `rules` in the root and package configs and `--rule <id>=<severity>`, values `error | warning | off` (`off` drops the issue). Ids may be patterns (`represent/*`). For an issue the sources are consulted flags > its package's config > root config; the first source with a match decides; within a source an exact id beats a pattern, then the longer literal prefix wins. **Unknown keys** (02b Q4, C6 P14 kept): a key or pattern in a CLI-reserved namespace is always checked against the CLI's table; a key in any other namespace is checked against the module's `rules` when it declares them and accepted silently when it doesn't. A key that fails the check prints a warning, not an issue.
- **Exit codes and `--strict` (D164).** `check` exits non-zero when any `error` remains after overrides (D16), and on warnings too with the `warnings` part (`--strict=warnings` or config `strict`). `build` always runs the check and fails only on the `--strict` parts selected (`check`, `tests`, `unavailable`, `unhandled`, `warnings`), after writing its output. Failures outside scope never count; with `--no-tests`, the parts `tests`, `unavailable` and `unhandled` don't apply. `dev` never fails on issues. This replaces D76's `--no-check` and D8's `--fail-on-test-failure`.

## 10. Config: a CLI detail (D66), handed over to C6

**Not part of `-api`.** The config is user-facing and consumed only by the CLI: no plugin ever sees it (a module receives only its validated options; runners, severity overrides and directory configs are handled inside the CLI). So, like the check report (D60), it lives in the CLI package `@ofzza/tsshowcase` and is documented in the CLI README: users write `import { defineConfig } from '@ofzza/tsshowcase'` and point `$schema` at `./node_modules/@ofzza/tsshowcase/config.schema.json`. Plain `tsshowcase.config.json` stays fully supported (D6); `defineConfig` is optional typing sugar for `.ts`/`.js` configs. C6 implements it; the design agreed so far is kept here as C6's starting point:

**The JSON Schema is the source of truth (D58).** The schemas are **plain JSON files**, `config.schema.json`, `package-config.schema.json` and, since B9, `directory-config.schema.json` (C6: in the `config-schema/` unit, copied to the package root by the CLI's build for `$schema` paths, B5), each with a committed tsschema sidecar (`*.d.json.ts`) that gives the JSON import its literal types. The CLI package exports them as values, and the TS types below are **inferred** from them with tsschema, never written by hand. Documentation lives in each property's `description`. The shapes the inferred types must have (B9 changes marked):

```ts
export interface TsshowcaseConfig {
  module?: string; // reference (D9); default 'react-default'
  moduleOptions?: { [key: string]: unknown }; // validated by the module's optionsSchema
  themes?: string | readonly string[]; // references, optionally '#name' (§7); default ['default-light', 'default-dark']
  defaultTheme?: string; // a theme name
  outDir?: string; // build output, relative to the config file; default './showcase'; `build --out` (relative to the cwd) wins (B9, D167)
  scope?: readonly string[]; // B9, D165: paths and globs relative to each package; the default for every package; default ['.']
  ignore?: readonly string[]; // D103, D107: gitignore syntax incl. `!`, relative to the root (e.g. '/fixtures/'); overrides .gitignore/.tsshowcaseignore on the same path
  rules?: { [ruleIdOrPattern: string]: Severity | 'off' }; // B9, D176: ids or patterns
  testRunner?: string | readonly string[]; // B9, D172: pins the runner(s); skips detection (root config only)
  testRunners?: readonly string[]; // extra candidates (root config only)
  strict?: boolean | readonly StrictPart[]; // B9, D164: 'check' | 'tests' | 'unavailable' | 'unhandled' | 'warnings' (root config only)
  tests?: boolean; // false = --no-tests (D197)
  directories?: { [dir: string]: DirectoryConfig | true }; // B9, D188: keys relative to the repo root
  kinds?: { [fileAndName: string]: SymbolKind }; // B9, D189: literal '<path>#<name>' keys, relative to the repo root
  exempt?: { [pathOrGlob: string]: readonly ('docs' | 'tests' | 'showcase' | '*')[] }; // B9, D191
  vcs?: { webUrl?: string; directory?: string; host?: VcsInfo['host']; fileUrlTemplate?: string }; // D24, D110: override package.json#repository; no git
  dev?: { host?: string; port?: number }; // D113: --host / --port win; unset = the module/toolchain default (root config only)
  // removed by B9 (D164, D172, D192): check, failOnTestFailure, moduleOptions.testRunner
}

/** A workspace package's own tsshowcase.config.*: only package-scoped keys; repo-wide keys there are an error. */
export interface PackageConfig {
  skip?: boolean; // D69: leave the package out entirely (dropped from RepoInput.packages, its directories from the scan)
  scope?: readonly string[]; // B9, D165: overrides the root default for this package
  rules?: { [ruleIdOrPattern: string]: Severity | 'off' }; // consulted before the root rules (D176)
  moduleOptions?: { [key: string]: unknown }; // only per-package-marked keys are merged (D148)
  directories?: { [dir: string]: DirectoryConfig | true }; // B9: keys relative to the package
  kinds?: { [fileAndName: string]: SymbolKind }; // B9: relative to the package
  exempt?: { [pathOrGlob: string]: readonly ('docs' | 'tests' | 'showcase' | '*')[] }; // B9
}

/** B9, D188: a directory config, as a `.tsshowcase*` file (value: this object or `true`) or a `directories` entry. */
export interface DirectoryConfig {
  paths?: readonly string[]; // what the node lists: files, 'file#name', globs, resources (primary)
  name?: string;
  slug?: string; // URL-safe pattern in the schema
  order?: number;
  kinds?: { [fileAndName: string]: SymbolKind }; // relative to this directory; literal keys
  collections?: { [key: string]: readonly string[] }; // members matched like paths (D190); directory configs only
  exempt?: { [pathOrGlob: string]: readonly ('docs' | 'tests' | 'showcase' | '*')[] }; // relative to this directory
}

export type StrictPart = 'check' | 'tests' | 'unavailable' | 'unhandled' | 'warnings'; // D164

export function defineConfig(config: TsshowcaseConfig): TsshowcaseConfig; // identity, for typing in tsshowcase.config.ts
export function definePackageConfig(config: PackageConfig): PackageConfig; // identity, for a package's tsshowcase.config.ts (C6 P6)
export function defineDirectoryConfig(config: DirectoryConfig | true): DirectoryConfig | true; // B9: identity, for .tsshowcase.ts
```

- CLI flags override file values (D6). C6 validates config files with ajv against these schemas; normalisation (e.g. `themes: string` → `[string]`) and checks JSON Schema can't express (valid semver ranges, resolvable references, kind values known only once the module is loaded) happen in code after validation.
- **Directory configs (B9, D188):** `.tsshowcase` (JSONC) or `.tsshowcase.{json,jsonc,js,mjs,ts,mts}` in in-scope directories; several in one directory give `config/several-configs`, and none of them applies. Per directory, the sources merge key by key: the root config's `directories` entry < the package config's top level (package directory only: `kinds`, `exempt`) < the package config's `directories` entry < the file, with `config/shadowed` on overlap. A directory config creates a node unless it sets only `kinds`, `collections` and `exempt`. `paths` semantics, pattern rules and their issues: D188, D190 and C6.
- The JSON files are **published as they are**, as extra `exports` subpaths of the CLI package (`@ofzza/tsshowcase/config.schema.json`, `…/package-config.schema.json`, `…/directory-config.schema.json`), so editors get completion via `$schema`, and the CLI README links them. No emit step.
- Sidecars must be copied into `dist`, or consumers' inferred types silently become `never`: see C1 amendment A1 (D62).
- **Known gap (temporary):** until tsschema infers records (`additionalProperties`) and tuples, map-shaped fields (`rules`, `moduleOptions`, `directories`, `kinds`, `exempt`, `collections`) infer as `{ [key: string]: unknown }`. We accept that: the schemas (and so runtime validation) are already exact. Code that reads those maps (C6, C9) narrows at the point of use. Must be resolved before `1.0.0` (D80).

## 11. Theme catalogue slot (`catalogue/`, D28)

C2 provides the **mechanism**; C3 fills in the names and documents the hierarchy. **What C2's implementation ships (02b Fix 7b; C2 is implemented before C3, D194):** the shapes below with the two classes C2's own inclusion rules need (`tss-content`, `tss-theme-switcher`), `cssVar`, and the prefix test; C3 replaces the slot with its metadata object and adds everything else, its docs generation included. **B1:** filled by the C3 plan §8: one `as const` metadata object (`catalogue`) from which `tssVar`, `tssClass` and **`tssData`** (ARIA/data attribute names and values for states and variants, D84) are derived; the reference docs are generated into this area's README (D83). The sketch below shows the shape only.

```ts
export const tssClass = {
  content: 'tss-content', // README content container, needed by inclusion rule 6
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

A unit test asserts every class starts with `tss-` and every variable with `--tss-` (D45). C3 adds a test that the docs list exactly the exported names (`ci:catalogue`, D83).

## 12. Tests and contract guards

- **Runtime areas** (`version`, `issues`, `catalogue`, `inclusion`): ordinary tests, named after the files they test.
- **Type tests (`*.test-d.ts`)** in `module/`, `testrunner/`, `theme/`:
  - a minimal module / runner / theme set compiles against the constructor types via `satisfies`;
  - a module declaring a kind in `kinds` type-checks; `RouteTarget.anchor` without `doc` is allowed by the type but documented as invalid (no type trick); `Tests.status` and the open unions accept unknown strings;
  - **principle 2 guard:** a recursive, **test-only** `AssertJsonSafe<T>` type (rejects functions, `Map`, `Set`, `Date`; allows readonly arrays and optional members; treats the open-union `string & {}` as `string`; class instances can't be told apart structurally, so they are a code-review concern, 02b Fix 6a) applied to `Representation` and `TestsUpdate` (B9). A plain `JsonValue` assignability check can't work: interfaces have no index signature, readonly arrays aren't `JsonValue[]`, and optional members include `undefined`.
- **Compat guard (D13, D80):** `src/compat/v0.1.test-d.ts` holds frozen minimal implementations of a module, a runner and a theme set, written against the current contract. Each **reads** every context member it relies on (`ctx.tree`, `ctx.repo.files`, `file.read()`, `options.signal`, …), so removing or renaming one breaks it too; the negative fixture is a `@ts-expect-error` assignment showing a frozen module is not assignable to `ShowcaseModule & { newRequired(): void }` (02b Fix 6b). A change that would break them (a new required member, a narrowed type) fails CI. Before 1.0, a deliberate breaking change bumps the `-api` minor and replaces the guard file (`v0.2.test-d.ts`); from 1.0, files for later minors are added next to it and never edited (additive-only).
- **README samples:** every area README carries a type-checked sample (D56); the `module/` README sample is a complete, minimal module (an `analyze` returning one file with one symbol, plus `resolveRoute`); the `testrunner/` sample and the compat guard's runner implement `listTestFiles` and follow B7's `TestRunOptions`.
- **Before the first publish (D80),** contract changes such as B9 are made in place: the `v0.1` guard is written once, against the contract as implemented.

## 13. Documentation

- Each area README documents every exporting file in its area (the single place for that fact, D56, D163).
- The `-api` package README: purpose, the export list, the versioning rules (§2), and a "where to start" for module, runner and theme authors. The full authoring guides are C13.
- `CONTRIBUTE.md` glossary gains (B9, from 08b §1.1): symbol, exporting file, base, resource, showcasing component, association (primary/strong/weak), leftover, node (direct/indirect/root/home), configured node, own nodes, upward listing, directory config, collection, source, in scope, literal, walk order, tree order, covering (a module notion); plus analysis, issue, rule, inclusion rules, route target, theme set.
- AGENTS.md trigger table: "changed a contract" → update the area README; after the first publish, add an `-api` changeset (minor for breaking changes before 1.0) and update the compat guard (§12).

## Acceptance criteria

1. `npm run ci` passes with the `-api` areas, their READMEs, runtime tests and type tests.
2. `-api`'s only `dependency` is `@ofzza/tsschema`.
3. The README samples type-check: the minimal module (a valid `ShowcaseModuleConstructor`, `satisfies` included, with an `analyze` and a `resolveRoute`), the minimal runner (with `listTestFiles`), the minimal theme set and the composition example.
4. The compat guard is a permanent negative type test: a `@ts-expect-error` fixture proves that adding a required member to `ShowcaseModule` would break the frozen implementations.
5. `AssertJsonSafe<Representation>` and `AssertJsonSafe<TestsUpdate>` hold (B9), and a negative test proves the guard rejects a function, a `Map` and a `Date`.
6. Runtime tests: `IMPLEMENTED_API_VERSION` equals `-api`'s `package.json` version; `issue.error` / `issue.warning` produce the documented shape; the inclusion constants have their documented values; every catalogue class starts with `tss-` and every variable with `--tss-`.

## Recorded for later chunks

- **C9:** the `check --format json` report shape (`reportVersion`, tool info, summary, repo-level issues, the in-scope directory tree with its nodes, per-node status and issues; B9), exported as types from the CLI package and documented in the CLI README. The round-1 sketch is in this file's git history.
- ✅ **C3:** theme CSS is written unscoped and the CLI scopes it (done: C3 plan §5, D82); theme extension is composition (D65).
- **C11:** writing `BuildResult.files` (D64): path validation (relative, no `..`), the checkable inclusion rules (`index.html` present, no absolute build-machine paths), the output guard, an atomic swap into `outDir`, the marker file, and the written-files summary.
- **C15:** `-base-vite` builds with `build.write: false` and maps the Vite build output (chunks and assets; Rolldown in Vite 8) to `BuildFile[]`.
- **C5:** text docs (`DocRef.format: 'text'`, D71) render as escaped `<pre>` with a catalogue class; link resolution per §5.6 (D183); doc ids and prefixed element ids (D184); `RenderedDoc.headings` (B9).
- **C6/C11:** `dev` host and port flags and config keys (K8).
- **C6:** runner loading for the CLI (candidates incl. the module's static `testRunners`, pinning with `testRunner`, every detecting runner; D172, replacing D74's `loadTestRunner`); `optionFlags` parsing, precedence, collisions and `--help` (D79); module `kinds` collisions as load errors (D162); the directory-config schema and `defineDirectoryConfig` (B9).
- **C11:** `check` before `build`/`dev` and `--strict` (D164, replacing D76's `--no-check`); the CLI-run tests and `TestsUpdate` streaming with the shared revision (D172, D186, replacing D75); lazy docs, `DataUpdate` snapshots (also for content-only doc edits) and asset copying for rendered docs (D187).
- **C9:** the representation builder (D179, replacing the hierarchy helper and D78's group READMEs), associations (D173), the check (D174), issue placement (D180), the call order of §5.9.
- **C6:** per-package options (D69): merge each package's `moduleOptions` over the root ones and validate every merged result against `optionsSchema`.
- **C6:** the config (§10, D66) moves into the CLI package: `config.schema.json` / `package-config.schema.json` with sidecars, the inferred `TsshowcaseConfig` / `PackageConfig` types, `defineConfig`, the `exports` subpaths, and docs in the CLI README. Acceptance criteria carried over from C2: `dist` contains every `*.d.json.ts` sidecar next to its `.json`, and a consumer-style type test over `dist` infers `TsshowcaseConfig` as an object type, not `never`; the published `config.schema.json` validates a sample `tsshowcase.config.json` with ajv and rejects an invalid one. Also: ajv (draft 2020-12, `useDefaults`) for config and module options; readable error formatting.
- **C15 (B9):** `prepareData(Representation)`, showcase loaders and `loadShowcase(file, exportName)`, `applyTestUpdate` over `TestsUpdate` with the §4.4 merge table, `DataUpdate` with `state`, the lazy-docs mechanics (embedding rendered HTML), serving the pre-copied `docAssetsDir` (02b Q2).
- **C3 (C3-A3, B9):** the catalogue state attribute for broken links (§5.6), `tssData` values for symbol kinds and test status; the theme-composition sample's check (02b Fix 7a).
- **C7 (B9):** `listTestFiles(options: TestListOptions)` returning `TestListing`, with the process guard and abort handling.
- **C4 (B9):** building the `ScannedTree` (roles at scan time, walk order, in scope).
- **C4, C5, C6, C9, C11 (Skip 1 in 02b):** they implement the behaviour C2 specifies in §4–§6 and §9; their plans reference those sections rather than restating them.
- **C2's implementation (Skip 2 in 02b):** verify the tsschema export names behind `JsonSchema`/`OptionsOf` and decide the "defaulted properties are required" transform (a tsschema feature or a local mapped type), with a type test; consider making `VcsInfo.kind` an open union or removing it. **C6 (plan 7):** move `ResolvedTheme` into the CLI (no `-api` contract uses it since D102).
- **Before `1.0.0` (after C14):** tsschema record/tuple inference must have landed, and the `unknown` gap in §10 closed.
- **C14:** first publish is `0.1.0` (D80); the PR changeset check is switched on after it; PUBLISH.md documents the `0.x` major-changeset gotcha; whether `0.x` releases also use a `next` dist-tag is decided there. The pack-and-install smoke test also checks the sidecars reach the tarball and that a consumer's `defineConfig` call type-checks.

## Out of scope

Loading and version checks (C6), the catalogue contents (C3), any implementation of the contracts (C4–C11), authoring guides (C13).

## Open questions

- ✅ **Q1 Schema technology:** JSON Schema (draft 2020-12) for module options and config, ajv for validation in the CLI, `@ofzza/tsschema` to infer the TS types. Record/tuple inference is on tsschema's roadmap; temporary `unknown`s are acceptable until `1.0.0` (§10, D80). Round 3: the schemas are plain `.json` files with tsschema sidecars, not `as const` literals (D58, D62).
- ✅ **Q2 Showcases per unit:** an array (`showcases[]`). *B9: showcases per (file, export) on nodes (D195).*
- ✅ **Q3 Per-package config in workspaces:** per-package config files (the user's preference, familiar from tsconfig/.gitignore). Only in workspace package directories in v1, with a restricted `PackageConfig` schema; repo-wide keys there are an error. Nested directory configs could be added later, additively. *B9: directory configs added (D188).*
- ✅ **Q4 `CheckReport`:** a CLI detail, not in `-api`. `-api` holds only the internal plugin contracts; the report is external output, consumed only by the CLI, documented in the CLI README, with a `reportVersion` field. Specified in C9.
- ✅ **Q5 Required methods (D63):** only `analyze` is required; everything else is optional and feature-detected per action, including `verify`. The user's argument: a stub `verify` is as bad as a fake `build`. *B9: `verify` is per file and no longer needed for `check`; `runTests` removed (D177, D198).*
- ✅ **Q6 Dev data updates (D68):** full snapshots, with an increasing `revision` on snapshots and test updates so stale messages can be dropped. Patches can be added later, additively. *B9: one revision shared by snapshots and test updates (D186).*
- ✅ **Q7 `build` and `check` (D76):** `check` is a prerequisite of `build` and `dev`; check errors fail `build` by default (`--no-check` opts out); `dev` shows issues but never fails on them. (Claude first proposed the opposite; the user's question turned it around.) *B9: replaced by `--strict` parts (D164).*
- ✅ **Q8 Severity overrides applied by the CLI** (D70), which derives an issue's package when unset. *B9: patterns and precedence (D176); placement (D180).*
- ✅ **Q9 `resolveRoute` signature (D67):** `resolveRoute(target, ctx, heading?)` with the tagged `RouteTarget`; routes stored in the data. *B9: `resolveRoute(target, ctx)` with `RouteTarget { node, doc?, anchor? }` (D183).*
- ✅ **Q10 Versioning (D80):** every package starts at `0.1.0`; no changesets before the first publish; pre-1.0, minors may break; additive-only from `1.0.0`. (The user's revision of the `1.0.0` proposal.)
