# Representation redesign (from C8 planning)

> Status: **revision 3.1, DRAFT for review** (2026-10-20). Revision 1 was approved on 2026-10-19 and reviewed (`08c-redesign-review.md`, decisions X1–X13, clarifications Y1–Y16); revision 2 folded those in and was reviewed by four focused reviewers (`08d-redesign-review.md`, decisions Z1–Z18, clarifications V1–V12); revision 3 folded in everything decided since and was reviewed for clarity and self-sufficiency (`08e-redesign-review.md`, decisions E1–E8, fixes F1–F9). This revision applies E1–E7 and F1–F9; the choices it made beyond them were reviewed one by one with the user (08e, G1–G8, all settled). Nothing here is applied to the approved plans yet; §11 lists every touched decision with its verdict and §12 the affected plans, in application order. Once approved, the decisions are recorded in AGENTS.md (D160+) and the plans revised one by one (§12); after that this document becomes history and the plans and D160+ are authoritative (E6).

## 0. Principles

1. **Stages with clear interfaces first.** Each stage of the pipeline (§3) must be serviceable, not perfect. Rules *inside* a stage (how resources associate, which issues are raised, defaults) may be refined later, once real examples run, as long as the interfaces between stages stay stable.
2. **The target is the user's own repositories, mainly new ones**, built to conventions the user is happy to live with. Sane, practical requirements matter; making every existing repository layout work does not.
3. **Presentation belongs to the module.** The CLI provides facts (symbols, associations, nodes, test results, issues) in a defined default order; what is shown, hidden, merged or summarised is the module's choice (coverage, test summaries, home-doc selection). *(Drawn from the outcomes of Z1, Z3 and X7.)*

## 1. Why

Pre-C8, one concept, the **unit** (a directory, D36), carried three roles at once: what is checked (README, tests, showcase per unit), what gets a menu item, and what gets a page. The menu tree was derived from directories by the CLI (C2 §4.1). In C8, every case where those roles needed to differ led to another redefinition of "unit" or "group".

The redesign separates the concerns:

- **What exists:** the scanned, in-scope file tree, enriched with roles (docs, tests, showcases, public symbols).
- **What is checked:** every public symbol must be **represented** with the resources its kind requires.
- **What is presented:** a tree of **representable nodes**, created from exporting files and explicit configuration; the module decides how nodes become pages and menus.

### 1.1 Glossary

- **Symbol:** a publicly exported declaration, identified by its declaring file and declared name (`file#name`, split on the last `#`).
- **Exporting file:** a file that declares at least one symbol. A **collection** (§8.4) stands in for one exporting file.
- **Base:** a file's name minus its last extension (`Button.tsx` → `Button`, `Button.types.ts` → `Button.types`); `.d.ts`, `.d.mts` and `.d.cts` count as one extension (`index.d.ts` → `index`). A collection's key is its base, used as is.
- **Resource:** a doc, test or showcase file. A **showcasing component** is an export of a showcase file that the module recognises as a showcase.
- **Association:** a link from a resource to a node (through an exporting file's or collection's direct node, or directly), at one of three levels: **primary > strong > weak** (§4.3).
- **Leftover:** a resource directly in a directory that has neither a by-name nor an explicit association. Home docs are never leftovers.
- **Node:** a `RepresentableNode` (§7): **direct** (for an exporting file or collection), **indirect** (for a configured directory), **root** (a package's root), or **home** (repo or package root documents).
- **Configured node:** a node created by a directory config: an indirect node, or the root when a directory config merged into it (`configuredBy`, §7.2).
- **A directory's own node:** the node whose `path` equals the directory (an indirect node there, or the root for the package directory and for its `configuredBy` directory).
- **A file's own nodes:** its direct node (or its collection's node) and every configured node listing it, except upward listings (§7.7).
- **Upward listing:** a `paths` entry that lists a file not below the node's directory. Only those entries are ignored when choosing link targets; they still count for the check (X6).
- **Directory config:** a `.tsshowcase*` file, or an entry of a config file's `directories` map (§8.2); type `DirectoryConfig` (§8.1).
- **Source** (for the check): an exporting file with its associations, or a node listing a symbol (§5.1).
- **In scope:** a directory or file at or below a match of its package's `scope` (§3.2), below the package directory and not below a nested package, and not ignored (D103, D109) or skipped. Directories outside every package are never in scope.
- **Literal:** a pattern with no unescaped glob characters.
- **Walk order:** a directory's files before its subdirectories, names sorted in UTF-16 code units, recursively (§3.1); identical on every machine.
- **Tree order:** depth-first over the node tree, children in their final order (§7.5); homes come before the tree.
- **Covering:** a module notion (not part of the contract): a configured node showing symbols that also have their own nodes (§7.6).

## 2. Requirement changes

REQUIREMENT.md stays verbatim; these are recorded as decisions (as D37, D40 were), one decision per RQ (§12). All agreed.

| # | Before | Now |
| - | ------ | --- |
| RQ1 | Every public export must "be located in its own directory" (D36) | Every public symbol needs **representation**: by its exporting file's own node, or by a configured node; several symbols per node are fine. |
| RQ2 | "Component, service or utility"; D37, D38 | **Types are named explicitly.** Requirements per symbol kind (§5.2): docs for every kind; tests for components, hooks, functions, classes; showcases for components, hooks. The CLI defines them for the well-known kinds; a module declares the kinds it adds and their requirements; the CLI checks. Modules can't change well-known kinds' requirements. |
| RQ3 | "Have its own README.md" | **Documentation** is markdown associated with the symbol (§4.3); a directory `README` documents every exporting file in its directory. |
| RQ4 | D76 (`--no-check`), D8 (`--fail-on-test-failure`) | **`--strict` with parts** (§9.1); `build` always runs `check` and fails only on the selected parts (or unrecoverable errors). `--no-check`, `check: false`, `--fail-on-test-failure`, `failOnTestFailure` removed; `--rule` added. `dev` never fails on issues; `check` exits non-zero on errors (D16). |
| RQ5 | The report shows "all the not-ignored subdirectories"; `scope` root-relative, root-only (D101, D107) | Only **in-scope** directories, `scope` **package-relative** (§3.2); directories outside packages are never in scope. |
| RQ6 | The module verifies per directory (D2, D21) and runs tests (D3, D34, D42) | The **CLI** checks representation (§5) and owns test runners (§4.2); the module enriches and keeps an optional **per-file `verify`** (§6). |
| RQ7 | `build [outDir]` (D93) | **Actions never take optional positional arguments**; `build --out <dir>` (§9.3). |

## 3. The pipeline

```
1. scan (CLI)                    in-scope file tree per package (§3.1); directory configs (§8)
2. enrichments                   CLI: docs, configs; test files via the test runners (§4.2)
                                 module: AnalysisResult (§4.1): symbols, showcasing components, issues, inputs, state, meta
3. associations (CLI)            resources → direct nodes of exporting files / collections, and → configured nodes (§4.3)
4. representation (CLI)          homes, package roots, node trees (§7)
5. check (CLI)                   every symbol represented (§5) → issues
6. verify (module, optional)     framework-specific issues per file (§6)
7. issues attached to nodes      and printed (stdout / --format json) (§7.3)
8. build/dev only                tests run (CLI) → Representation.tests (§7.9); routes (resolveRoute, §7.7);
                                 module build/dev with lazy doc rendering (§7.10)
```

- **`check`** runs steps 1–7. **`init`** runs steps 1–4 and scaffolds (§10). **`build`/`dev`** run everything; `--strict` decides whether issues and test outcomes fail `build` (§9.1).
- Kind overrides (§8.3) are applied by the CLI after step 2, before associations; everything from step 3 on sees effective kinds.

### 3.1 The scan

- Produces, per package, the **in-scope file tree** (`ScannedTree`, below), with lazy, cached, tracked reads through `RepoFiles` (D40, D108).
- Unchanged: ignored = nonexistent (D103, D108, D109), hard ignores, workspace detection (C4), PnP refusal.
- Repo and package metadata (manifests, home docs, licences, the main configs) are read wherever they are (D101). **Directory configs are read only in scope** (§8.2).
- A package's tree **stops at nested package directories** (C4's deepest-package rule).
- Walk order: a directory's **files before its subdirectories**, names sorted in UTF-16 code units (C4), identical on every machine.
- Analysis tools (the TypeScript compiler) read entries and barrels from disk wherever they are; only their *results* are limited to in-scope files.

```ts
interface ScannedTree {
  packages: ScannedPackage[]; // in walk order of their directories
}

interface ScannedPackage {
  dir: RepoPath; // the package directory ('.' in a single-package repo)
  root: ScannedDirectory; // the package directory, pruned to in-scope directories and the ancestors connecting them
}

interface ScannedDirectory {
  path: RepoPath;
  inScope: boolean; // false only for ancestors kept to connect in-scope directories to the package directory
  files: ScannedFile[]; // in-scope files only (empty when !inScope), in walk order
  directories: ScannedDirectory[]; // in walk order
}

interface ScannedFile {
  path: RepoPath;
  name: string;
  roles: FileRole[]; // which roles are present depends on the stage: §3.1
}

type FileRole = 'exporting' | 'doc' | 'test' | 'showcase' | 'config'; // open union
```

- **Roles:** `doc` = a `.md` file, or a `README`/`LICEN[CS]E`/`CHANGELOG` in a non-markdown form where §4.3 and §7.8 accept one; `config` = a directory config file; `test` = listed by a runner (§4.2); `exporting` = at least one symbol (§4.1); `showcase` = at least one showcasing component. A file may have several roles; `test` and `showcase` are exclusive (a file that is both counts as a test, §4.3).
- **Roles depend on the stage (G6; documented on `AnalyzeContext.tree` and `ScannedFile.roles`):** the tree in `AnalyzeContext` carries only the CLI's scan-time roles, `doc` and `config`. `exporting` and `showcase` are what `analyze` itself produces, and `test` is left out so that the CLI can list test files **in parallel** with the analysis; a module that needs to know about tests matches file names itself. Every later context (verify, scaffold, build, dev) gets a tree with every role.

### 3.2 `scope`, package-relative

- `scope` lists paths and globs **relative to each package** (in a single-package repo, the repo root). Default `['.']`. A match includes everything below it.
- Set in the root config as the default for every package, overridable in a package config. `--scope` applies to every package; one "unmatched" warning per entry, printed only when the entry matches in no package; an unmatched config `scope` entry gets the same warning. A printed warning, not a check issue (G5: it concerns the run's arguments or settings, not the repository's structure).
- The workspace root is never a package (C2 §4.3); a repo with no manifest has no packages and nothing in scope. Running on one package: `skip`/`ignore` (documented). `scope: ['src']` is recommended for typical layouts (documented).

## 4. Enrichments and associations

### 4.1 Module analysis (Z4)

```ts
interface AnalyzeContext {
  repo: RepoInput;
  tree: ScannedTree; // roles: only 'doc' and 'config' (exporting/showcase come from this analysis; tests are listed in parallel), §3.1
  signal: AbortSignal; // dev aborts a stale analysis
}

interface RepoInput {
  root: string; // absolute
  name: string;
  info?: PackageInfo;
  vcs?: VcsInfo;
  layout: 'single' | 'workspace';
  packages: PackageInput[];
  files: RepoFiles; // D40, D108
}

interface PackageInput {
  dir: RepoPath;
  info: PackageInfo;
  scope: string[]; // the package's effective scope entries
  options: unknown; // D69, D148: root moduleOptions merged with this package's marked ones, validated
}

interface AnalysisResult {
  files: {
    [path: RepoPath]: {
      symbols?: FileSymbol[]; // in declaration order
      showcasingComponents?: ShowcasingComponent[];
      meta?: JsonObject; // module data for this file
    };
  };
  issues?: Issue[]; // package- or repo-level, or about one file (placement: §7.3)
  inputs?: RepoPath[]; // dev re-analysis triggers (D68)
  state?: unknown; // handed back to verify / scaffold / resolveRoute / build / dev; never serialised
  meta?: JsonObject; // becomes Representation.meta
}

interface FileSymbol {
  name: string; // declared name ('default' when the declaration has no identifier)
  kind: SymbolKind; // 'component' | 'hook' | 'function' | 'class' | 'constant' | 'type' | module-declared (open union)
  meta?: JsonObject; // copied verbatim to RepresentedItem.meta
}

interface ShowcasingComponent {
  name: string; // identity: the export name ('default' for the default export); unique per file
  meta?: JsonObject; // copied verbatim to ShowcaseAssociation.meta (e.g. the `shell` opt-in, a title)
}
```

- A symbol attaches to the file that **declares** it (its value declaration's file; type-only symbols to their first declaration outside a module augmentation); being attached means it is publicly exported. One symbol per (file, name); with declaration merging the value meaning decides the kind, `type` only without one; namespaces holding only types are `type`. A JSON module's default export attaches to the `.json` file. Exported alias names are deferred (§13).
- Several showcasing components per file; identity = export name. A showcase file counts only if the module found at least one showcasing component in it.
- Declarations outside the package, in `node_modules` or in ignored files aren't attached; the module may report them through `issues`.
- In-source tests (Vitest `includeSource`) are both test files and exporting files and associate with themselves.
- **Validation (replaces D70's unit checks):** `files` keys outside scope are dropped silently; a key naming a nonexistent file, or two keys normalising to the same path, is a thrown error naming the module. A `FileSymbol.kind` that is neither well-known nor declared by the module (§5.2) is a `module/error` issue and the symbol is treated as `constant`.

### 4.2 CLI enrichments and test runners (X1, Z5, Z13, Z17)

- **Docs:** markdown files (role and format recorded); `README` in any form (§4.3).
- **Configs:** directory configs and config maps (§8).
- **Tests: the CLI owns test runners entirely**; modules never touch tests.
  - **Candidates:** the module's static `testRunners` (references resolved with the module's file as the parent, D9/D114 otherwise unchanged), then the built-ins, then config `testRunners`; de-duplicated by resolved identity, keeping the first position.
  - **Pinning:** config `testRunner` (one reference or a list) uses exactly those runners and skips `detect`.
  - **Detection:** otherwise every candidate whose `detect` resolves `true` is used. A `detect` that throws gives `tests/runner-failed` and the runner isn't used.
  - **Runner ids:** the runner's display name, de-duplicated in candidate order as `name`, `name-2`, …
  - **Listing:** `listTestFiles` (below) per runner, a **required** runner member. The union of listed files, **intersected with the in-scope tree**, gives the `test` role; listed files outside it are only counted per runner (`outOfScope`). A file listed by several runners runs with each.
  - **Runs** are unfiltered (every test the runner finds), then intersected with the in-scope tree like listings. Files a run reports without having listed them are added to the results and attached **weakly** (§7.4); they are never re-checked. In `dev`, listing is redone on structure changes.
  - **Failures:** a runner whose listing returns `status: 'error'` (or rejects), or whose run fails, gives `tests/runner-failed`; the other runners' results are used. **Any** listing failure suppresses every `represent/missing-tests` issue; a listing's `failedRoots` suppress it for symbols declared under those directories.
  - **No runner** while at least one symbol requires tests: a repo-level `tests/no-runner` error; every `represent/missing-tests` suppressed; no fallback pattern.
  - **No symbol requires tests** (after kind overrides, module-declared kinds and exemptions): the requirement is short-circuited, no `tests/no-runner` and no `--strict` failure for a missing runner (`tests.status = 'not-run'`, reason "no tests required"); detected runners still list and run existing tests (e.g. type tests).
  - The runner restores `process.env` and process handlers around listing too (`createVitest` itself changes them). Vitest 3.x/4.x resolve inline project `root`s against the process cwd: documented only (Z15).

```ts
interface TestRunner {
  detect(repo: RunnerRepoInput): MaybePromise<boolean>; // unchanged (C2 §6)
  listTestFiles(repo: RunnerRepoInput): Promise<TestListing>; // new, required
  run(options: TestRunOptions): Promise<RawTestReport>; // unchanged; the CLI calls it without a files filter
  watch(options: TestRunOptions, onUpdate: (update: RawTestUpdate) => void): Promise<RawTestReport>; // unchanged
}

interface TestListing {
  status: 'completed' | 'error'; // 'error' = listing failed entirely (never throws where it can, like run)
  error?: TestError;
  files: { file: RepoPath; project?: string }[]; // repo-relative; identity (file, project)
  failedRoots?: { dir: RepoPath; error: TestError }[]; // directories where listing failed while others worked (D135's shape)
}
```

### 4.3 Associations (X2, Y6, Z8, Z16, E1)

| Rule | Level |
| ---- | ----- |
| **By name:** a resource named `<base>.` + anything belongs to the exporting file (or collection) with base `<base>` in the same directory; the **longest matching base wins**; two exporting files with the same base and different extensions are a `config/same-base` error (their resources stay unassociated); exact case (D130). Name-matched docs are `<base>.md` only. | **primary** |
| **Explicit:** listed by a configured node's `paths` or a collection's resource members (§8) | **primary** |
| **Directory README** (`README`, `README.md`, `README.txt`, case-insensitive; among several, markdown first, then name order; non-markdown renders as text; Z16, confirmed by the user): **primary** for the directory's configured node (incl. the root when its `configuredBy` lies **below** the package directory, E1) | **primary** |
| … and **strong** for every exporting file in that directory, whether or not a configured node exists | **strong** |
| **Single exporting file:** with exactly one exporting file (or collection) in a directory, its leftovers belong to it | **strong** |
| **Unmatched:** attached to the nearest root or indirect node above by path (incl. the directory's own node) | **weak** |

- A resource reached by several routes keeps its highest level once per target.
- **Home documents** (§7.8) take no part in associations: a package-root or repo-root README is not a directory README for these rules and never a leftover. The exception is an explicit listing: a home doc listed in a configured node's `paths` (or a collection) is also associated (primary) there; in particular, when a directory config merges into the root at the package directory, the package README becomes primary for the root only if that config lists it (E1).
- A file that is both a test and a showcase counts as a test.
- A module may skip a strong doc on a file node when a parent node already shows it as primary (presentation, Z8).
- Convention (documented): name tests and showcases after the file they cover.

## 5. The check (CLI)

### 5.1 Rule

For every symbol, a **source** is its exporting file (with its primary and strong associations) or any node listing it, wherever it sits (upward listings count, X6), with that node's primary and strong resources. The symbol passes if **at least one source** has **every** resource type its kind requires (after exemptions).

- Only **primary** and **strong** count; **weak** never satisfies anything.
- **One issue per symbol and missing type**, naming the direct source and the configured nodes considered: rule ids `represent/missing-docs`, `represent/missing-tests`, `represent/missing-showcase` (§5.3).
- Listing a symbol vouches for it (Z11): a broad listing (`paths: ["**"]` with one doc, test and showcase) satisfies everything below it; documented prominently.
- The `check` summary reports how many symbols are exempted (one total, §8.5).

### 5.2 Requirements per kind (defined by the CLI, documented)

| Kind | Docs | Tests | Showcase |
| ---- | ---- | ----- | -------- |
| `component`, `hook` | required | required | required |
| `function`, `class` | required | required | optional |
| `constant`, `type` | required | optional | optional |
| module-declared kinds | `static kinds: { [kind]: { requires: { docs?, tests?, showcase?: 'required' \| 'optional' } } }`; an omitted `requires.*` is optional; unknown resource types are ignored (new resource types arrive later as new optional arrays on nodes, V3) | | |

- A module kind colliding with a well-known one is a load error naming the module.
- Kind overrides (§8.3) apply first; exemptions (§8.5) remove requirements per symbol.

### 5.3 Rule ids and default severities (E5)

Schema violations of configs and directory configs stay thrown `ConfigError`s (C6). Everything below is a check issue; every severity is overridable (§5.4). C6 validates `rules` keys against this table plus the module's declared rules.

| Rule id | Default | Raised when |
| ------- | ------- | ----------- |
| `represent/missing-docs`, `represent/missing-tests`, `represent/missing-showcase` | error | no source of a symbol has the required resource type (§5.1) |
| `tests/no-runner` | error | no runner while tests are required (§4.2) |
| `tests/runner-failed` | error | a runner's `detect` throws, or its listing or run fails (§4.2) |
| `config/several-configs` | error | several directory config files in one directory |
| `config/collection-clash` | error | a collection key clashes with a real non-member file or a directory (§8.4) |
| `config/collection-membership` | error | an exporting file matched by several collections (§8.4) |
| `config/kind-conflict` | error | several competing non-ancestor kind overrides (§8.3) |
| `config/same-base` | error | two exporting files in a directory with the same base (§4.3) |
| `config/unused-key` | warning | a `kinds` or `exempt` key that matches no symbol, or a `directories` key naming no in-scope directory |
| `config/missing-path` | warning | a listed path, collection member or `#name` that doesn't exist; a glob matching nothing |
| `config/no-role` | warning | a literal `paths` entry or collection member that is no exporting file, doc, test or showcase (a literal directory: hint "use `sub/**`") |
| `config/out-of-scope` | warning | an entry, key or path pointing out of scope or into another package (the entry is ignored) |
| `config/shadowed` | warning | a directory-config key set by several sources for one directory (§8.2); a kind override shadowed by a nearer one (§8.3) |
| `config/slug` | warning | duplicate sibling slugs |
| `docs/broken-link`, `docs/broken-anchor` | warning | §7.7 (D123) |
| `module/error` | error | invalid module output that isn't fatal (e.g. an undeclared kind, §4.1) |
| module-declared (`react/*`, `headless/*`, …) | the module's | the module's analysis or `verify` |

Where each issue attaches: §7.3.

### 5.4 Severity overrides

- `rules` in the root config and package configs, and `--rule <id>=<severity>` (§9.2); values `error | warning | off` (`off` drops the issue).
- Rule ids may be **patterns** (`represent/*`). For an issue, the sources are consulted in the order **flags > the issue's package config > root config**; the first source with a matching entry decides. Within a source, an exact id beats a pattern, then the pattern with the longer literal prefix wins.
- A `rules` key or `--rule` id (exact or pattern) matching no known rule id gives a warning (printed, not an issue).

## 6. Module `verify` (per file, optional)

```ts
verify?(file: FileInput, ctx: VerifyContext): MaybePromise<Issue[]>;

interface VerifyContext {
  repo: RepoInput;
  tree: ScannedTree; // every role (§3.1)
  representation: Representation; // tests.status 'not-run', reason "check" (§7.9)
  package: PackageInput; // the file's package, with its options
  state: unknown; // from AnalysisResult.state
}

interface FileInput {
  path: RepoPath;
  package: RepoPath;
  roles: FileRole[];
  symbols: RepresentedItem[]; // effective kinds (after overrides)
  showcasingComponents: ShowcasingComponent[];
  meta?: JsonObject; // from AnalysisResult.files[path].meta
  associations: { node: string; level: 'primary' | 'strong' | 'weak'; type: 'doc' | 'test' | 'showcase' }[]; // as a resource
  nodes: string[]; // the file's own nodes, else [the nearest node above]: where its issues attach (§7.3)
  read(): Promise<string>; // lazy, cached, tracked (D40)
}
```

- Called for **every in-scope file**. No `ignored` verdict. The CLI forces each returned issue's `file` to the input file.
- **Invalidation in `dev`:** content edits re-verify the files whose tracked reads changed; changes to enrichments, associations, configs or the representation re-verify every file.

## 7. The representation

### 7.1 `RepresentableNode`

```ts
interface RepresentableNode {
  id: string; // opaque (meaning via path and representation)
  representation: 'direct' | 'indirect' | 'root' | 'home'; // open union
  path: RepoPath; // the file (direct), the virtual '<dir>/<key>' (collection), or the directory (indirect, root, home)
  configuredBy?: RepoPath; // root: the directory whose config merged into it (Z7)
  name?: string; // from configuration; modules derive a default ('index.*' → its directory's name)
  slug?: string; // from configuration (URL-safe; duplicate sibling slugs: config/slug)
  order?: number; // from configuration
  route?: string; // from resolveRoute (build/dev, D67)
  items: RepresentedItem[]; // in final order
  docs: DocRef[]; // in final order
  tests: Association[]; // in final order
  showcases: ShowcaseAssociation[]; // in final order
  issues: Issue[];
  children: RepresentableNode[]; // in final order; always [] for homes
  meta?: JsonObject;
}

interface RepresentedItem { file: RepoPath; name: string; kind: SymbolKind; meta?: JsonObject }
interface Association { file: RepoPath; association: 'primary' | 'strong' | 'weak'; ordinal: number } // open union; ordinal per node and type
interface ShowcaseAssociation extends Association { name: string; meta?: JsonObject }
interface DocRef { path: RepoPath; id: string; format: 'markdown' | 'text'; role?: 'readme' | 'licence' | 'changelog' | 'other'; association?: 'primary' | 'strong' | 'weak'; ordinal: number }
```

- **Ids:** opaque strings; homes `home:<dir>`, roots `root:<package dir>`, others path-based in v1, but modules never parse them.
- **Order (Z2):** the CLI emits every array in its final default order (§7.5); `order`/`ordinal` stay as information (`ordinal` = the index in the default order); modules may re-sort.
- **Doc ids (E4):** `doc-` + the first 8 hex digits of the sha256 of the doc's repo path (`doc-3f2a9c1e`). When two docs of one representation collide, the colliding docs take the shortest longer prefix (in steps of 4 hex digits) that makes them distinct. Element ids inside a rendered doc are prefixed `<doc-id>--` (`doc-3f2a9c1e--props`).
- **Roles:** `readme` for README forms (home and directory READMEs), `licence` for `LICEN[CS]E` forms, `changelog` for `CHANGELOG` (any case, `.md` or no extension), `other` for the remaining home docs; other docs carry no role.

### 7.2 Building the tree (per package)

1. **Direct nodes:** one per exporting file (or collection), holding its symbols and its primary and strong associations; path = the file's path (a collection: `<dir>/<key>`, §8.4).
2. **Indirect nodes:** one per directory config that creates a node (§8.2), holding what its `paths` list and its directory's README (primary); path = the directory.
3. **Root (Z7):** always id `root:<package dir>`, representation `'root'`, path = the package directory. A configured node **merges into** the root (its `name`, `slug`, `order`, `paths`, its README per §4.3) when it sits at the package directory, or at the scope path when the package's `scope` has exactly one entry and that entry is literal; recorded as `configuredBy`.
4. **Nesting:** a node's parent is the node whose path is the nearest ancestor path of its own; nodes without one are children of the root. Collections nest by their directory. The tree is sparse.
5. **Duplicates are allowed:** a symbol listed by several nodes appears in each; modules decide what to show.
6. **Upward listings** sit at their own directory and repeat symbols; they count as check sources and are ignored when choosing link targets.

### 7.3 Issue placement (F5)

| Issue | Attached to |
| ----- | ----------- |
| symbol issues (`represent/*`) | every source considered: the symbol's direct node and each configured node listing it |
| doc issues (`docs/*`) | every node holding the linking doc's `DocRef` |
| config issues (`config/*`) | the config directory's own node, else the nearest node above; root-config issues are repo-level |
| `verify` issues, and module analysis issues with an in-scope `file` | the file's own nodes, else the nearest node above (`FileInput.nodes`) |
| other module analysis issues with a `package` | that package's root |
| repo-level issues (incl. `tests/*`) | `Representation.issues` and the repo home's `issues` |

- "The nearest node above" is a root or indirect node; homes never take issues or resources that way (they only hold home docs and, for the repo home, repo-level issues).
- `Issue` loses `dir` (no directory verdicts); the CLI derives `package` from `file` when unset. All issues are printed in the report; whether a module shows them is its choice.

### 7.4 Weak associations

An unassociated resource attaches **weakly** to the nearest root or indirect node above it by path, including its directory's own node, never to a sibling file's direct node or a home. Weak associations exist for presentation only; this includes tests (a failing internal test stays visible). Modules may hide weak tests by default (C10).

### 7.5 Ordering

- **Items:** a direct node's items in declaration order (the order of `AnalysisResult` symbols); a configured node's items by listed order (files ordered by the first `paths` entry matching them, matches of one glob in walk order), each file's symbols in declaration order.
- **Resources (docs, tests, showcases):** listed resources first in listed order (a resource matched by several entries ordered by the first), then the rest in walk order (§3.1).
- **Children:** nodes with `order` first (ascending), then walk order by path; a collection sorts as a file named `<key>` in its directory.
- **Representation:** homes before the tree; packages in walk order of their directories.
- Finer rules are deferred (§13).

### 7.6 Coverage (module concern, Z1)

Not part of the contract. The data holds what a module needs (nodes' `items`, `path`, the tree). `-base-react` defines its own notion (e.g. an indirect node covering the symbols of descendant direct nodes it lists; upward listings never cover). Per-node test summaries are likewise computed by modules from `tests` associations and `Representation.tests`.

### 7.7 Links (Z9, E3)

The CLI decides **which node** a link targets; the URL comes from the module's `resolveRoute(target, ctx)`:

```ts
resolveRoute?(target: RouteTarget, ctx: { representation: Representation; state: unknown }): string | undefined;

interface RouteTarget {
  node: string; // node id
  doc?: RepoPath; // set iff the link target is a doc in the selected node's docs
  anchor?: string; // only with doc: the rendered, doc-id-prefixed element id
}
```

Static rules, independent of routes (D124 holds):

0. **Target:** a file link is explicit; a directory link means its `README` if it has one (an implicit file link), else the directory itself.
1. **Nonexistent or ignored** → broken.
2. **A home doc, or a repo or package root directory** → its home (a package's home; the repo home for the repo root).
3. **Out of scope, or outside every package** → the VCS host.
4. **Candidates and selection:**
   - a file: first **its own nodes**, else the nodes it is **associated** with;
   - a directory without README: its own node only;
   - selection: strongest association first (primary > strong > weak; a file's own nodes count as primary), then shallowest tree depth, then tree order;
   - no candidates: a file → the VCS host (D73); a directory → broken.

- **`resolveRoute` returning `undefined`** → the VCS host. A module may answer for a node it doesn't present with another node's URL (e.g. a covering node).
- **VCS host links** use `VcsInfo.fileUrlTemplate` (D110), fragments passed through unchanged. With no VCS info, such a link renders as an `<a>` without `href` and raises no issue (the one-time `build` warning covers it, D110).
- **Broken links** (nonexistent, ignored, not represented): rendered as an `<a>` **without `href`**, with a catalogue state attribute for themes, plus `docs/broken-link` (revises D121).
- **Fragments:** element ids of every doc (headings, footnotes, author anchors) are prefixed with the doc's id (§7.1); a fragment on a doc target is rewritten to the prefixed id and passed as `anchor`; on other in-app targets it is dropped. A fragment that doesn't exist in the target doc gives `docs/broken-anchor`. Authors write GitHub-style `#props` (D127, D128 parity).
- **Permalinks:** a doc's own headings link to the page of its **link-selection node** (the node a link to the doc file would select).

### 7.8 Homes (X7, Z3)

- The repo `home` and each package's `home` hold the root's documents: **every markdown file** directly in that root, plus `README`, `LICEN[CS]E` and `CHANGELOG` with no extension (and `README`/`LICEN[CS]E` as `.txt`), case-insensitive, non-markdown rendered as text (D71); `DocRef`s with a `role` (§7.1).
- **The module selects** what to show (`react-default`: README and licence by default; a `homeDocs` option taking patterns or roles for more).
- **Single-package repos:** the package's home is the repo home (same id `home:.`); `layout` says which case applies; the module decides whether to merge a package's home with its root.

### 7.9 Tests in the data (X9, Z5, Z6)

```ts
interface Tests {
  status: 'ran' | 'not-run' | 'unavailable'; // open union
  reason?: string; // not-run: "--no-tests", "no tests required", "check"; unavailable: "no runner"
  runners: {
    [id: string]: { // opaque CLI-assigned id (§4.2)
      name: string;
      reference?: string; // only when not a filesystem path
      status: 'completed' | 'error';
      error?: TestError;
      failedRoots?: { dir: RepoPath; error: TestError }[];
      unhandledErrors?: TestError[];
      outOfScope?: number; // listed or run files outside the in-scope tree (Z17)
      results: { [file: RepoPath]: TestFileResult[] }; // in-scope files only; one per project, `project` kept
    };
  };
}

interface TestsUpdate { // dev only
  revision: number; // shared with data snapshots (Z6)
  status?: Tests['status']; // present when it changed
  reason?: string;
  runners: {
    [id: string]: { // only runners with changes; the runner set is fixed per dev session
      status?: 'completed' | 'error';
      error?: TestError;
      results?: { [file: RepoPath]: TestFileResult[] }; // replace per (file, project)
      removed?: { file: RepoPath; project?: string }[];
      failedRoots?: { dir: RepoPath; error: TestError }[]; // replaces when present
      unhandledErrors?: TestError[]; // replaces when present (an empty array clears)
      outOfScope?: number;
    };
  };
}
```

- **Status:** `ran` when at least one runner ran (whatever its outcome); `not-run` with `--no-tests`, when nothing is required and no runner exists, and during `check` (which lists but never runs); `unavailable` when no runner exists while tests are required.
- Stored once; nodes reference test files through `tests: Association[]`.
- **`dev`:** one monotonic `revision` shared by data snapshots and test updates; a module applies a message only if its revision is higher than the last applied one.

### 7.10 `Representation` and lazy docs (X7, V2, E2)

```ts
interface Representation {
  apiVersion: string;
  revision: number; // shared with test updates (Z6)
  generatedAt: string;
  layout: 'single' | 'workspace';
  home: RepresentableNode & { name: string; info?: PackageInfo; vcs?: VcsInfo }; // id 'home:.'; name: root info.name, else the directory name
  packages: { dir: RepoPath; info: PackageInfo; home: RepresentableNode; root: RepresentableNode }[];
  tests: Tests;
  issues: Issue[]; // repo-level
  meta?: JsonObject;
}

interface DocsAccess {
  render(ref: DocRef): Promise<RenderedDoc>; // cached, bound to the snapshot the ref came from
  read(ref: DocRef): Promise<string>; // the raw source
}

interface RenderedDoc {
  html: string; // catalogue classes, prefixed element ids, rewritten links and media (C5)
  headings: DocHeading[]; // ids prefixed (§7.1); empty for text docs
  assets: DocAsset[]; // D120; empty for text docs
}
```

- The data stays plain JSON (C2 principle 2). Docs are `DocRef`s. `BuildContext` and `DevContext` gain `representation` (replacing `data: ShowcaseData`) and `docs: DocsAccess`; `DevContext` gains `onDocsUpdate(listener: (update: { revision: number; docs: DocRef[] }) => void)` for re-rendered docs; `onDataUpdate` delivers `Representation` and `onTestUpdate` delivers `TestsUpdate`. Other members as in C2 §5.8.
- **The interface is fixed here and in C2 (E2).** The mechanics (how rendered HTML is embedded in the bundle, how the dev docs update travels with the shared revision, render tracking for asset copying) are designed in the C15 revision. Binding constraint: **only rendered docs and their assets reach the output**; a doc's assets are written iff `render` was called for it in this build; in `dev`, only rendered docs are re-rendered on change.
- `check` still validates the links and anchors of **every** `DocRef`, home docs included (D123); that needs parsing, not rendering.

### 7.11 `ScaffoldContext`

```ts
interface ScaffoldContext {
  repo: RepoInput;
  tree: ScannedTree; // every role
  analysis: AnalysisResult;
  representation: Representation; // init runs steps 1–4
  state: unknown;
  target: RepoPath; // `tsshowcase init <dir>`
}
```

`ScaffoldResult` and `ScaffoldFile` are unchanged (C2 §5.10).

## 8. Configuration

### 8.1 Shapes

```ts
type DirectoryConfigValue = DirectoryConfig | true; // true = {}; false is a schema error

interface DirectoryConfig {
  paths?: string[]; // §8.2
  name?: string;
  slug?: string; // URL-safe pattern in the schema
  order?: number;
  kinds?: { [fileAndName: string]: SymbolKind }; // §8.3; literal keys
  collections?: { [key: string]: string[] }; // §8.4; directory configs only
  exempt?: { [pathOrGlob: string]: ('docs' | 'tests' | 'showcase' | '*')[] }; // §8.5
}

// Root and package configs (C6), the redesign's keys only:
interface RedesignConfigKeys {
  scope?: string[]; // §3.2; package-relative
  directories?: { [dir: string]: DirectoryConfigValue }; // keys relative to the repo root (root config) or the package (package config)
  kinds?: DirectoryConfig['kinds']; // top level counts as a source at the config's directory
  exempt?: DirectoryConfig['exempt'];
  testRunner?: string | string[]; // root config only; pins runners (§4.2)
  testRunners?: string[]; // root config only; extra candidates
  strict?: boolean | StrictPart[]; // root config only (§9.1)
  rules?: { [idOrPattern: string]: 'error' | 'warning' | 'off' };
  // `collections` is not allowed at the top level of root and package configs (schema error)
}
```

### 8.2 Directory configs and `directories`

- **Files:** `.tsshowcase` (JSONC; empty = `{}`) or `.tsshowcase.{json,jsonc,js,mjs,ts,mts}` (default-exported value, `defineDirectoryConfig()`), **in any in-scope directory**; several in one directory: `config/several-configs`, none of them applies. TypeScript's wildcard `include`s skip dotfiles (spike), so they stay out of `tsc` builds; `npm pack` publishes them when their directory is in `files`.
- **In config files:** the `directories` map in the root and package configs (an exception to D112 like `scope`/`ignore`). An entry for a directory that isn't in scope: `config/out-of-scope`.
- **Precedence for one directory (F6):** the root config's `directories` entry < the package config's top level (package directory only: `kinds`, `exempt`) < the package config's `directories` entry < the directory's own file. Each key of a later source **replaces that key** of the earlier ones wholesale (no deep merge), with a `config/shadowed` warning per key defined in more than one source. (Replaces D154's "one directory configured twice is an error".)
- **Creates a node** unless all its keys are non-node keys (`kinds`, `collections`, `exempt`): empty, `{}`, `true`, display-only and `paths` configs all create an indirect node; no `paths` = `paths: []` (a grouping node). New keys declare their side.
- **`paths`:** what the node lists, primary (§4.3): symbols by file, `file#name` or glob, and docs, tests and showcases. Listing an exporting file doesn't bring its name-matched resources along. Entries:
  - a literal exporting file lists all its symbols; `file#name` one symbol (`#name` literal, an unknown name: `config/missing-path`);
  - a collection key lists the collection; a collection member lists only that member's own symbols;
  - a literal directory: `config/no-role` with the hint "use `sub/**`";
  - a glob lists every match; a glob matching nothing: `config/missing-path`;
  - may reach upward within scope (upward listings, §1.1); never into another package (`config/out-of-scope`, ignored).
- **Patterns:** picomatch relative to the config's directory, `dot: true`, globs match files, no negations in v1 (a leading `!` is rejected), `#name` split off before matching, `../` only as a literal prefix within scope; directory names with glob characters must be escaped in globs (documented); patterns the CLI builds are escaped.
- **Warnings:** a listed path that doesn't exist (`config/missing-path`); a **literal** entry that isn't an exporting file, doc, test or showcase (`config/no-role`); anything pointing out of scope (`config/out-of-scope`, X10).

### 8.3 Kind overrides

- `kinds: { '<path>#<name>': kind }`, a top-level key of the root and package configs (relative to the repo root, the package) and in directory configs and `directories` entries (relative to that directory). Keys are literal (no globs); `#default` names an anonymous default export.
- Applied by the CLI after analysis (§3).
- **Nearest wins:** ancestor configs of the declaring file compete first, the deepest winning (sources at one directory are merged first, §8.2; a config's top-level `kinds` counts as at its directory). Non-ancestor overrides count only when no ancestor sets the kind: exactly one wins; several are a `config/kind-conflict` and no override applies. A shadowed override gives `config/shadowed`; unused keys give `config/unused-key`.
- Values: the well-known kinds plus the module's declared kinds; anything else is a `ConfigError` once the module is loaded.

### 8.4 Collections

- A **stand-in for one exporting file**, declared in a directory config (`collections: { [key]: members }`): exporting members' symbols merge into one exporting "file" named by the key; the key is its base (used as is) for name matching; members' own name matches are added (union); members never compete with their own collection.
- **Members are matched exactly like `paths` entries** (G3): plain names, paths and globs relative to the collection's directory, within scope and the package. Every exporting file a member matches becomes an **exporting member** (its symbols merge into the collection), in subdirectories too (`**`); every doc, test or showcase it matches becomes a **resource member** (primary; a resource may be in several collections). A literal member with neither role gives `config/no-role`; a missing one, or a glob matching nothing, `config/missing-path`. A new file matched by an existing glob joins the collection; narrower globs are the remedy for unwanted matches (documented).
- An exporting file matched by several collections (named or by glob): `config/collection-membership`, it stands alone.
- The collection's node: path `<dir>/<key>` (virtual), sorting as a file `<key>` (§7.5); members get no direct nodes. A key clashing with a real non-member file or a directory: `config/collection-clash`, the collection isn't created.
- References to members (`paths`, `kinds`, `exempt`) keep naming the real files; the single-file rule counts a collection as one file.

### 8.5 Exemptions (X8)

- `exempt: { '<path or glob>[#name]': ('docs' | 'tests' | 'showcase' | '*')[] }`, same places and relativity as `kinds`; the only key with globs.
- Matching entries from every location combine (union); unused keys give `config/unused-key`; the `check` summary reports how many symbols are exempted (one total).

### 8.6 Other config changes

- `testRunner` and `testRunners` are CLI keys (§4.2); `strict` (§9.1); `rules` accept patterns (§5.4); `outDir` stays (relative to the config file).
- Removed: `check`, `failOnTestFailure`, `units`; `moduleOptions.testRunner` (no longer a module option).

## 9. Command line

### 9.1 `--strict` (X13, Z13, Z14, F6)

| Part | Fails `build` when |
| ---- | ------------------ |
| `check` | any check error remains after severity overrides |
| `tests` | any in-scope test fails (incl. file- and suite-level errors) |
| `unavailable` | tests couldn't run although required: no runner, a runner's listing or run failed, `failedRoots` |
| `unhandled` | a runner reported `unhandledErrors` |
| `warnings` | any check warning remains (not part of a bare `--strict`) |

- `--strict` alone selects `check`, `tests`, `unavailable`, `unhandled`; parts are selected **only** in the `=` form, repeated (`--strict=tests --strict=check`); bare and `=` forms combine as a union. The argv normaliser turns a bare `--strict` into the default parts and handles `--no-strict` (none, for the run). Empty or unknown part names are rejected after parsing.
- Config: `strict: true | false | Part[]` (`false` and `[]` select none). Any `--strict`/`--no-strict` flag replaces the config value.
- Failures outside scope never count. With `--no-tests`, the parts `tests`, `unavailable` and `unhandled` don't apply.
- `build` writes its output first, then exits non-zero on a strict failure.
- `check` honours the `warnings` part (from flags or config `strict`): non-zero exit on warnings; other parts are ignored by `check`, which exits non-zero on errors anyway.

### 9.2 `--rule`

`--rule <id>=<severity>` (repeatable; `error | warning | off`; split on the last `=`; an id starting with `-` uses `--rule=…`), on top of the root and package `rules` (§5.4); ids may be patterns (`represent/*`).

### 9.3 No optional positionals

Actions never take optional positional arguments (required ones, like `init <dir>`, are fine). `build --out <dir>` (relative to the cwd) > config `outDir` (relative to the config file) > `./showcase`. With no positionals, `build --strict tests` is an unambiguous error ("`build` takes no positional arguments; did you mean `--strict=tests`?"). Migrating from an existing `showcase/` app: set `--out`/`outDir` (documented).

### 9.4 `--no-tests`

`build` and `dev` skip running tests (`tests.status = 'not-run'`, reason "--no-tests"); `check` still lists test files (associations need them).

## 10. `init`

Runs steps 1–4 (§3) and calls `scaffold` with the representation (§7.11). Re-expressed in the new model during the C8 revision (from D150): for an exporting file or collection missing resources, create the missing stubs by name (`<base>.md`, `<base>.test.*`, `<base>.showcase.*`) as its kinds require, respecting exemptions; targeting a directory or a new unit directory creates `README.md` there; a new item gets its file, stubs and the entry export. Details (targets, collections, no-entry packages) are part of the revised C8 plan.

## 11. Decision verdicts

### 11.1 C8 rounds 1–2 (D149–D159)

| Decision | Verdict |
| -------- | ------- |
| D149 showcase file format | **Changed:** several showcasing components per file, identity = export name, `shell` and titles through `ShowcasingComponent.meta` |
| D150 `init` | **Changed** (§10) |
| D151 `react/showcase-in-build` | **Kept** (a module issue through `AnalysisResult.issues`, since it reads tsconfig/`files`) |
| D152 TypeScript selection | **Kept** |
| D153 units by declaring directory and claims | **Replaced** (§7.2, §8.2) |
| D154 directory configs | **Changed:** `units` → `directories`; claims → `paths`; in scope only; type `DirectoryConfig`; one directory configured by several sources is merged key by key with `config/shadowed` warnings instead of an error (§8.2); several config files in one directory stay an error (`config/several-configs`) |
| D155 `showcase: false` | **Replaced** (no node; `exempt` for the check) |
| D156 `name`, `slug`, `order` | **Changed:** on nodes; slugs validated at check time (`config/slug`, schema pattern) |
| D157 broadened groups | **Replaced** (indirect nodes, weak associations) |
| D158 kind overrides by declaration | **Changed:** nearest wins; `config/kind-conflict` only for competing non-ancestor overrides; unused-key warning kept |
| D159 `headless` preset | **Kept** as module `verify` rules |

### 11.2 Earlier decisions touched

| Decision | Verdict |
| -------- | ------- |
| D2, D21, D40, D108 (`DirectoryInput`) | **Replaced:** CLI check + per-file `verify`; `RepoFiles` and its access rule (D108) stay |
| D3, D26 (test-result mapping), D34, D42, D74, D75, D135, D136 | **Replaced:** the CLI owns runners and maps results (§4.2) |
| D5 | **Changed:** analysis = enrichments (`AnalysisResult`, §4.1) |
| D8, D76 | **Replaced** by `--strict` parts (§9.1) |
| D9, D114 | **Changed:** module `testRunners` resolve with the module's file as the parent |
| D16 | **Kept**; report over in-scope directories |
| D19 | **Re-expressed** with homes (§7.8) |
| D23 | **Changed:** lazy `docs.render`/`docs.read` |
| D29 | **Changed:** C7 before C9 (Z12) |
| D36, D37, D38 | **Replaced** (RQ1, RQ2) |
| D41 | **Changed:** dogfooding under the new rules (one-file units pass; multi-file units need collections or naming) |
| D43 | **Changed:** the CLI assembles `Representation` (not units, HTML, tests) |
| D59 | **Changed:** package configs gain `scope`, `directories`, `kinds`, `exempt` (`strict` and the runner keys are root-only) |
| D60 | **Kept:** the report stays a CLI detail; its shape (C9) follows the in-scope directories and nodes |
| D61, D96, D100, D140 | **Changed:** showcases per (file, export); `shell` per component via `meta`; loaders for every file with showcasing components |
| D63 | **Changed:** analysis required; `verify`, `resolveRoute`, `build`, `dev`, `scaffold`, `dispose` optional; `runTests` gone; statics `testRunners`, `kinds` |
| D67 | **Changed:** routes per node via a target object (§7.7) |
| D68 | **Changed:** `AnalysisResult.inputs` kept; one revision shared by snapshots and test updates |
| D69 | **Kept:** per-package options as `PackageInput.options`; `skip` unchanged |
| D70 | **Changed:** validation of enrichments (§4.1); issue placement (§7.3) |
| D72 | **Changed:** packages always present; a module may hide one |
| D73, D78, D121 | **Changed:** link rules (§7.7); broken links keep a link element without `href` |
| D91 | **Kept:** "the repo has React showcases" now means showcasing components were found |
| D93 | **Changed:** `build --out` (RQ7) |
| D101, D106, D107, D112 | **Changed:** package-relative `scope`; results limited to in-scope files; `directories`/`kinds`/`exempt` keys root- or package-relative |
| D111 | **Kept:** C6 parses argv (now with the `--strict` normaliser and `--rule`), C9 owns the command framework |
| D116, D145, D148 | **Wording:** `testRunner` is no longer a module option |
| D120, D125 | **Changed:** assets only for rendered docs |
| D123 | **Changed:** broken links and anchors stay check issues; the hierarchy helper is replaced by the representation builder; `checkDocs` covers every `DocRef` |
| D124 | **Kept:** whether a link is broken never depends on routes |
| D127, D128 | **Changed:** ids prefixed with the doc id |
| D132, D133, D134 | **Kept:** the runner's run plan and version handling; the CLI drives the runner, and listing uses the same plan |
| D141 | **Changed:** test update shape (§7.9) |
| H36 | **Replaced** by doc-id prefixing |
| U16 (08a) | **Reversed:** directory configs only in scope (X10) |

### 11.3 Untouched

Themes and isolation (C3, D81–D92, D95, D97), CLI writes (D64), doc-asset safety (D129, D130), ignore semantics (D103, D109), option marking mechanics (D116, D148), config files (D115), C15's Vite pipeline (D138, D139, D142–D147) apart from showcase loading and the data/test/doc update shapes, markdown rendering (D118, D119, D122, D126, D131) apart from link targets and id prefixing, versioning (D13, D80).

### 11.4 Open points resolved or obsolete

- C8: `testRunner` root-only text and marking (obsolete); 08a U1–U6 dissolved (U7, U8 carried to C8, §12).
- C9: the D106 drop order and the hierarchy helper (replaced by associations and the representation builder); analysis validation (now of enrichments).
- C10: `runTests` passing `{ repo }`, `failedRoots` → unavailable units, `queued` counted as running, runner call serialisation (all CLI-side now); H36 (prefixing).
- C11: whether "tests unavailable" fails strict mode (part `unavailable`); defaulting absent units; `onAnalysisUpdate`; `LinkTargets` from the hierarchy and group routes; out-of-scope groups (C4 M6); aborting `runTests` on SIGINT (now the CLI's runners).
- L2 (relativising `TestError` paths) clearly sits with the CLI (C11); L3 (`unhandledErrors` reach the data and `--strict`) resolved.

## 12. Effects on the existing plans

To be worked through after this revision is approved, **plan by plan in this order**, each presented, approved by the user and committed (E7): AGENTS decisions → C2 → C1, C3, C4, C5, C6, C7, C15 → super-plan R10 → C8. Amendment ids: C2 **B9** (the C8 round-2 draft's B9 is void), C1 **A4**, C3 **C3-A3**, a "redesign amendment" section in C4, C5, C6, C7 and C15, super-plan **R10** (the draft's R10 is void), decisions **D160+**. `08f-application-checklist.md` is a starting point; this document wins where they differ. Once all are applied, this document gets the banner "applied; the plans and D160+ are authoritative; kept as design rationale" and AGENTS "Current focus" is removed (E6).

- **AGENTS.md:**
  - §3: **D160+**, one decision per RQ (RQ1–RQ7) plus one per substantive outcome (draft list in 08f); the statuses of existing decisions per §11 ("replaced / changed by D1xx"), D149–D159 included;
  - §1: assignment summary annotations and "Architectural shifts" (representation, CLI-owned check and runners, `--strict`);
  - §2: the chunk tracker, "Current focus" (removed when done), the open points per §11.4 plus new items, a "Deferred" row from §13, the implementation order;
  - §4: L2 → C11, L3 resolved;
  - §6: the per-directory verification pitfalls superseded.
- **C2 (B9):**
  - design principles (2: `Representation`; 4 "directories are identities" removed; 6: the CLI owns check, associations, runners);
  - §1: the package layout under the new dogfood rules (as C1 below);
  - §3: `SymbolKind`, `JsonObject`;
  - §4: the data model replaced (`Representation`, `RepresentableNode`, `RepresentedItem`, `Association`, `ShowcaseAssociation`, `DocRef`, `Tests`, `TestsUpdate`, `RenderedDoc`; `PackageInfo`, `VcsInfo`, `DocAsset`, `DocHeading` kept; `ShowcaseData`, `RepoNode`, `PackageNode`, `UnitNode`/`GroupNode`, `Doc` removed);
  - §5.1: statics `testRunners`, `kinds`; §5.2: required/optional methods (`runTests` removed);
  - §5.3: `RepoFiles` stays, the stage-dependent roles documented on `AnalyzeContext.tree` and `ScannedFile.roles` (G6), `DirectoryInput` removed, `ScannedTree`, `FileInput`, `RepoInput`/`PackageInput` revised (`readme`/`licence` dropped, `scope` added), `RepoInput.overrides` removed;
  - §5.4: `AnalyzeContext`/`AnalysisResult`, attachment rules, the new validation (replacing D70's), `AnalyzedUnit`, `unitRoot`, `ExportedSymbol.entry/typeOnly` removed;
  - §5.5: per-file `verify`, `VerifyContext`; §5.6: `resolveRoute` with `RouteTarget`, the links subsection;
  - §5.7: replaced by a note (tests are the CLI's; `TestContext`, `loadTestRunner` removed);
  - §5.8: `BuildContext`/`DevContext` with `representation`, `docs`, `onDataUpdate`, `onTestUpdate`, `onDocsUpdate`, the assets rule;
  - §5.9: the call order from the pipeline (§3); §5.10: `ScaffoldContext` (§7.11); §5.11: the `testRunner` paragraph removed;
  - §6: `listTestFiles`, `TestListing`, process restore around listing, "the module maps" removed;
  - §8: rules 6 (prefixed ids), 7 (`shell` via `meta`), the embedding of `Representation`; inclusion rules unchanged in substance;
  - §9: the issue model (`dir` removed, rule namespaces and the rule-id table of §5.3, patterns, `--strict`, `module/error` per file);
  - §10: the config sketch (§8.1);
  - §12: guards (`AssertJsonSafe<Representation>`, `AssertJsonSafe<TestsUpdate>`, the compatibility guard replaced);
  - §13: glossary (§1.1); acceptance criteria; "Recorded for later chunks"; open questions annotated.
- **C1 (A4):**
  - §8, the new dogfood layout: each concept is one exporting file `packages/<pkg>/src/<area>/<Concept>.ts(x)`; one `README.md` per area directory documents every exporting file in it (strong); tests and showcases named after the file they cover (`<Concept>.test.ts(x)`, `<Concept>.showcase.tsx` where the kind requires one); a `.tsshowcase` in an area directory where a menu level is wanted; `scope: ['src']` in the root config; per-unit barrels (`<Unit>/index.ts`) and `_`-prefixed internal directories are no longer needed (non-exporting files need nothing); `src/index.ts` re-exports the declaring files; the build-tsconfig exclusions of tests and showcases stay;
  - §10: trigger-table rows (exporting file, area README, directory config, collection);
  - §5 and "Recorded for later chunks": `ignore: ['/fixtures/']` dropped, because `fixtures/` lies outside every package (the workspace root is never a package), so it is never in scope;
  - glossary seed (§1.1).
- **C3 (C3-A3):** `tssData` values (test status per node, symbol kinds; "Unit info" renamed), the state attribute for broken links; classes or attributes for association levels and node issues now or with C10.
- **C4 (redesign amendment):** the seam (`ScanOptions.scope` per package, `ScannedTree`, collected directory-config files, `toRepoInput`); package-relative scope, the per-entry unmatched warning, no package drop, the D106 paragraph removed; per-package trees, nested packages, walk order; `DirectoryInput` removed, tracking kept; homes with roles, README forms and preference; directory configs (in scope); recorded points.
- **C5 (redesign amendment):** link rules (§7.7, the E3 order); doc ids (E4) and prefixing; fragments; the permalink owner (link-selection node); `<a>` without `href` for broken and info-less VCS links; the callback `(target) => string | undefined`; the API (`LinkTargets` → the representation, lazy per-doc render returning `RenderedDoc`, `checkDocs` over every `DocRef`, assets per render); the hierarchy helper (§8.2) deleted; recorded points.
- **C6 (redesign amendment):** `ResolvedConfig` (`strict`, `rules` patterns, `--rule`, `--out`, runner loading, normalised directory configs); the argv normaliser (`--strict`, `--no-strict`) and `--rule` parsing; `--no-tests`; the schemas (root, package, `directory-config.schema.json`, `defineDirectoryConfig`, `collections` rejected at the top level); the precedence of §8.2; per-package merging; module `testRunners` resolution; runner loading rewritten (candidates, pinning, all detecting); the rule-id table (§5.3) for `rules` validation; "did you mean" for removed keys and the old positional.
- **C7 (redesign amendment):** `listTestFiles` returning `TestListing` (via `globTestSpecifications()`); several detected runners are no longer an error; restore around listing; the Vitest 3/4 inline-root caveat documented; the "independent of cwd" test scoped; README: `testRunner` is a CLI key; recorded points move to C9/C11.
- **C15 (redesign amendment):** `prepareData(Representation)`; showcase loaders keyed by file, `loadShowcase(file, exportName)`; `applyTestUpdate` over `Representation.tests` with `TestsUpdate`; one revision; the lazy-docs mechanics (E2: embedding rendered HTML, the dev docs update, render tracking for assets).
- **Super-plan (R10):** the revision note; package table rows; chunk texts (C1, C2, C4–C15); the C2 CLI-duties table rewritten (scan, enrichments, associations, representation, check, issue placement, runners, `--strict`, lazy docs, asset copying); edges C9 → C7 added, C10 → C7 dropped (C11 keeps C7, Z12); implementation order C1 → C2 → C4 → C6 → C7 → C8 → C9, then C3 → C5 → C15 → C10 → C11 → …
- **C8 (React base, rewritten):** `AnalysisResult` (symbols with kinds incl. the TS rules and the syntactic fallback, showcasing components with `meta`), per-file `verify` (`headless`, showcase format), `react/showcase-in-build` as an analysis issue, `scaffold` (§10), options (`homeDocs` is `react-default`'s; no `testRunner`); fixtures re-derived. From 08a: **carried** U7 (namespace re-exports, `export =`), U8/U18 (`init` with no entry; `init` targets, re-expressed without claims), U9–U13 (entries, dist → source, externals, ignored sources, TS API precision), U15 (slug validation only), U17 (`inputs`, re-verify), U19–U22; **obsolete** U1–U6, U14, U16, U18's claim parts, and U23 except: `directory-config.schema.json` in C6's published files and C14's smoke test, the editor `json.schemas` tip, C13 "directory configs execute code (JS/TS)", C12 end-to-end coverage (re-expressed: directory configs, collections, `paths`, `slug`, `order`, exemptions).
- **C9:** associations, the representation builder, the CLI check, the report over in-scope directories, issue placement (§7.3), `--rule`, `check --strict=warnings`.
- **C10:** rendering `Representation` (homes with role-based selection and `homeDocs`, roots, nodes, coverage as its own notion, association levels, weak tests collapsed by default, issues, tests and per-node summaries), routes per node, showcases per (file, export), `shell` via `meta`.
- **C11:** the pipeline (§3), the CLI-run tests and their `dev` streaming with the shared revision, `--strict` parts, `--no-tests`, lazy docs and asset copying, dev invalidation, `--out`, L2.
- **C12, C13, C14:** acceptance cases, docs (principles, conventions, the vouching effect, escaping, migration), the pack test.

## 13. Deferred

- Module-proposed collections (e.g. the `headless` preset) and glob keys in `directories` (Z10).
- Warnings for incomplete configured nodes.
- The import path (entry/subpath) of an item on its page; exported alias names.
- Finer ordering rules (children defaults, home docs).
- A module hint for preferred test runners; priority between pinned/module runners and defaults.
- Grouping issues per file in the report.
- A `vouch: false` key (Z11).

**Rejected for now:** a hint in `represent/missing-*` issues pointing at weakly attached resources of the missing type (Z18).
