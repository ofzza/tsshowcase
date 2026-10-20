# Representation redesign (from C8 planning)

> Status: **revision 3, DRAFT for review** (2026-10-20). Revision 1 was approved on 2026-10-19 and reviewed (`08c-redesign-review.md`, decisions X1–X13, clarifications Y1–Y16); revision 2 folded those in and was reviewed by four focused reviewers (`08d-redesign-review.md`, decisions Z1–Z18, clarifications V1–V12). This revision folds in everything decided and approved since. Nothing here is applied to the approved plans yet; §11 lists every touched decision with its verdict and §12 the affected plans. Once approved, the decisions are recorded in AGENTS.md (D160+) and the affected plans revised.

## 0. Principles

1. **Stages with clear interfaces first.** Each stage of the pipeline (§3) must be serviceable, not perfect. Rules *inside* a stage (how resources associate, which issues are raised, defaults) may be refined later, once real examples run, as long as the interfaces between stages stay stable.
2. **The target is the user's own repositories, mainly new ones**, built to conventions the user is happy to live with. Sane, practical requirements matter; making every existing repository layout work does not.
3. **Presentation belongs to the module.** The CLI provides facts (symbols, associations, nodes, test results, issues) in a defined default order; what is shown, hidden, merged or summarised is the module's choice (coverage, test summaries, home-doc selection).

## 1. Why

Pre-C8, one concept, the **unit** (a directory, D36), carried three roles at once: what is checked (README, tests, showcase per unit), what gets a menu item, and what gets a page. The menu tree was derived from directories by the CLI (C2 §4.1). In C8, every case where those roles needed to differ led to another redefinition of "unit" or "group".

The redesign separates the concerns:

- **What exists:** the scanned, in-scope file tree, enriched with roles (docs, tests, showcases, public symbols).
- **What is checked:** every public symbol must be **represented** with the resources its kind requires.
- **What is presented:** a tree of **representable nodes**, created from exporting files and explicit configuration; the module decides how nodes become pages and menus.

### 1.1 Glossary

- **Symbol:** a publicly exported declaration, identified by its declaring file and declared name (`file#name`, split on the last `#`).
- **Exporting file:** a file that declares at least one symbol. A **collection** (§8.3) stands in for one exporting file.
- **Resource:** a doc, test or showcase file. A **showcasing component** is an export of a showcase file that the module recognises as a showcase.
- **Association:** a link from a resource to an exporting file or a node, at one of three levels: **primary > strong > weak** (§4.3).
- **Node:** a `RepresentableNode` (§7): **direct** (for an exporting file or collection), **indirect** (for a configured directory), **root** (a package's root), or **home** (repo or package root documents).
- **Directory config:** a `.tsshowcase*` file, or an entry of a config file's `directories` map (§8.1); type `DirectoryConfig`.
- **Source** (for the check): an exporting file with its associations, or a node listing a symbol (§5.1).
- **Tree order:** depth-first over the node tree, children in their final order (§7.5).
- **Covering:** a module notion (not part of the contract): a configured node showing symbols that also have their own nodes (§7.6).

## 2. Requirement changes

REQUIREMENT.md stays verbatim; these are recorded as decisions (as D37, D40 were). All agreed.

| # | Before | Now |
| - | ------ | --- |
| RQ1 | Every public export must "be located in its own directory" (D36) | Every public symbol needs **representation**: by its exporting file's own node, or by a configured node; several symbols per node are fine. |
| RQ2 | "Component, service or utility"; D37, D38 | **Types are named explicitly.** Requirements per symbol kind (§5.2): docs for every kind; tests for components, hooks, functions, classes; showcases for components, hooks. The CLI defines them for the well-known kinds; a module declares the kinds it adds and their requirements; the CLI checks. Modules can't change well-known kinds' requirements. |
| RQ3 | "Have its own README.md" | **Documentation** is markdown associated with the symbol (§4.3); a directory `README` documents every exporting file in its directory. |
| RQ4 | D76 (`--no-check`), D8 (`--fail-on-test-failure`) | **`--strict` with parts** (§9); `build` always runs `check` and fails only on the selected parts (or unrecoverable errors). `--no-check`, `check: false`, `--fail-on-test-failure`, `failOnTestFailure` removed; `--rule` added. `dev` never fails on issues; `check` exits non-zero on errors (D16). |
| RQ5 | The report shows "all the not-ignored subdirectories"; `scope` root-relative, root-only (D101, D107) | Only **in-scope** directories, `scope` **package-relative** (§3.2); directories outside packages are never in scope. |
| RQ6 | The module verifies per directory (D2, D21) and runs tests (D3, D34, D42) | The **CLI** checks representation (§5) and owns test runners (§4.2); the module enriches and keeps an optional **per-file `verify`** (§6). |
| RQ7 | `build [outDir]` (D93) | **Actions never take optional positional arguments**; `build --out <dir>` (§9.3). |

## 3. The pipeline

```
1. scan (CLI)                    in-scope file tree per package; directory configs
2. enrichments                   CLI: docs, configs; test files via the test runners (§4.2)
                                 module: AnalysisResult (§4.1): symbols, showcasing components, collections?, issues, inputs, state, meta
3. associations (CLI)            resources → exporting files / collections / nodes (§4.3)
4. representation (CLI)          homes, package roots, node trees (§7)
5. check (CLI)                   every symbol represented (§5) → issues
6. verify (module, optional)     framework-specific issues per file (§6)
7. issues attached to nodes      and printed (stdout / --format json)
8. build/dev only                tests run (CLI) → Representation.tests (§7.9); routes (resolveRoute, §7.7);
                                 module build/dev with lazy doc rendering (§7.10)
```

- **`check`** runs steps 1–7. **`init`** runs steps 1–3 and scaffolds (§10). **`build`/`dev`** run everything; `--strict` decides whether issues and test outcomes fail `build` (§9).

### 3.1 The scan

- Produces, per package, the **in-scope file tree** (a typed tree of directories and files), with lazy, cached, tracked reads through `RepoFiles` (D40, D108).
- Unchanged: ignored = nonexistent (D103, D108, D109), hard ignores, workspace detection (C4), PnP refusal.
- Repo and package metadata (manifests, root docs, licences, the main configs) are read wherever they are (D101). **Directory configs are read only in scope** (§8.1).
- A package's tree **stops at nested package directories** (C4's deepest-package rule).
- Walk order: a directory's **files before its subdirectories**, names sorted in UTF-16 code units (C4), identical on every machine.
- Analysis tools (the TypeScript compiler) read entries and barrels from disk wherever they are; only their *results* are limited to in-scope files.

### 3.2 `scope`, package-relative

- `scope` lists paths and globs **relative to each package** (in a single-package repo, the repo root). Default `['.']`.
- Set in the root config as the default for every package, overridable in a package config. `--scope` applies to every package; one "unmatched" warning per entry.
- The workspace root is never a package (C2 §4.3); a repo with no manifest has no packages and nothing in scope. Running on one package: `skip`/`ignore` (documented). `scope: ['src']` is recommended for typical layouts (documented).

## 4. Enrichments and associations

### 4.1 Module analysis (Z4)

```ts
interface AnalysisResult {
  files: {
    [path: RepoPath]: {
      symbols?: FileSymbol[];
      showcasingComponents?: ShowcasingComponent[];
      meta?: JsonObject; // module data for this file
    };
  };
  issues?: Issue[]; // package- or repo-level (not about one in-scope file)
  inputs?: RepoPath[]; // dev re-analysis triggers (D68)
  state?: unknown; // handed back to verify / scaffold / resolveRoute; never serialised
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

- `AnalyzeContext` carries the scanned tree, `RepoFiles`, and each package's options (D69, D148).
- A symbol attaches to the file that **declares** it (its value declaration's file; type-only symbols to their first declaration outside a module augmentation); being attached means it is publicly exported. One symbol per (file, name); with declaration merging the value meaning decides the kind, `type` only without one; namespaces holding only types are `type`. A JSON module's default export attaches to the `.json` file. Exported alias names are deferred (§13).
- Several showcasing components per file; identity = export name. A showcase file counts only if the module found at least one showcasing component in it.
- Declarations outside the package, in `node_modules` or in ignored files aren't attached; the module may report them through `issues`.
- In-source tests (Vitest `includeSource`) are both test files and exporting files and associate with themselves.

### 4.2 CLI enrichments and test runners (X1, Z5, Z13, Z17)

- **Docs:** markdown files (role and format recorded); `README` in any form (§4.3).
- **Configs:** directory configs and config maps (§8).
- **Tests: the CLI owns test runners entirely**; modules never touch tests.
  - **Candidates:** the module's static `testRunners` (resolved from the module's own package), then built-ins and config `testRunners`, de-duplicated by resolved identity. **Pinning:** config `testRunner` (one or several) skips detection.
  - **Detection:** every candidate whose `detect` succeeds is used.
  - **Listing:** `listTestFiles(repo): Promise<{ file; project? }[]>` per runner; the union of listed files, **intersected with the in-scope, non-ignored tree** (results for other files are only counted per runner). Files a runner reports without listing them join the union (after the same intersection). A file listed by several runners runs with each. In `dev`, listing is redone on structure changes.
  - **Failures:** a runner that fails to run, or to list, is reported (`tests/runner-failed`); the others' results are used. A listing failure suppresses the per-symbol "missing tests" issues.
  - **No runner** while at least one symbol requires tests: a top-level `tests/no-runner` error; per-symbol "missing tests" suppressed; no fallback pattern.
  - **No symbol requires tests** (after kind overrides, module-declared kinds and exemptions): the requirement is short-circuited, no `tests/no-runner` and no `--strict` failure for a missing runner (`tests.status = 'not-run'`, reason "no tests required"); detected runners still list and run existing tests (e.g. type tests).
  - The runner restores `process.env` and process handlers around listing too (`createVitest` itself changes them). Vitest 3.x/4.x resolve inline project `root`s against the process cwd: documented only.

### 4.3 Associations (X2, Y6, Z8, Z16)

| Rule | Level |
| ---- | ----- |
| **By name:** a resource named `<base>.` + anything belongs to the exporting file with base `<base>` in the same directory; the **longest matching base wins**; two exporting files with the same base and different extensions are an error (their resources stay unassociated); exact case (D130). Name-matched docs are `<base>.md` only. | **primary** |
| **Explicit:** listed by a node's `paths` or a collection's resource members (§8) | **primary** |
| **Directory README** (`README`, `README.md`, `README.txt`; case-insensitive, an exact `README.md` preferred among several; non-markdown renders as text): **primary** for a configured node at that directory (incl. a root, Z7) | **primary** |
| … and **strong** for every exporting file in that directory, whether or not a configured node exists | **strong** |
| **Single exporting file:** with exactly one exporting file (or collection) in a directory, its other leftover resources belong to it | **strong** |
| **Unmatched:** attached to the nearest node above by path (incl. a node at its own directory) | **weak** |

- A resource reached by several routes keeps its highest level once per target.
- Home documents (§7.8) take no part, except when listed explicitly (then also primary there).
- A file that is both a test and a showcase counts as a test.
- Convention (documented): name tests and showcases after the file they cover.

## 5. The check (CLI)

### 5.1 Rule

For every symbol, a **source** is its exporting file (with its primary and strong associations) or any node listing it, wherever it sits (upward listings count, X6), with that node's primary and strong resources. The symbol passes if **at least one source** has **every** resource type its kind requires (after exemptions).

- Only **primary** and **strong** count; **weak** never satisfies anything.
- **One issue per symbol and missing type**, naming the direct source and the configured nodes considered: rule ids `represent/missing-docs`, `represent/missing-tests`, `represent/missing-showcase`; severities overridable (`rules`, `--rule`).
- Listing a symbol vouches for it (Z11): a broad listing (`paths: ["*/**"]` with one doc, test and showcase) satisfies everything below it; documented prominently.
- Kind-override conflicts are check issues; unused `kinds`/`exempt` keys are warnings.

### 5.2 Requirements per kind (defined by the CLI, documented)

| Kind | Docs | Tests | Showcase |
| ---- | ---- | ----- | -------- |
| `component`, `hook` | required | required | required |
| `function`, `class` | required | required | optional |
| `constant`, `type` | required | optional | optional |
| module-declared kinds | `static kinds: { [kind]: { requires: { docs?, tests?, showcase?: 'required' \| 'optional' } } }`; unknown resource types ignored | | |

Kind overrides (§8.2) apply first; exemptions (§8.4) remove requirements per symbol.

## 6. Module `verify` (per file, optional)

- `verify(file: FileInput, ctx: { tree, representation, state, package }): Issue[]`, called for **every in-scope file**; `FileInput` is the file with its enrichments and lazy contents; kinds in `ctx` are effective (after overrides).
- No `ignored` verdict. Issues of a file with no node attach to the nearest node above.
- **Invalidation in `dev`:** content edits re-verify the files whose tracked reads changed; changes to enrichments, associations, configs or the representation re-verify every file.

## 7. The representation

### 7.1 `RepresentableNode`

```ts
interface RepresentableNode {
  id: string; // opaque (meaning via path and representation)
  representation: 'direct' | 'indirect' | 'root' | 'home'; // open union
  path: RepoPath; // the file (direct) or directory (indirect, root, home)
  configuredBy?: RepoPath; // root: the directory whose config merged into it (Z7)
  name?: string; // from configuration; modules derive a default ('index.*' → its directory's name)
  slug?: string; // from configuration (URL-safe; duplicate sibling slugs warn)
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
- **Order (Z2):** the CLI emits every array in its final default order; `order`/`ordinal` stay as information; modules may re-sort.
- **Doc ids** are deterministic, HTML-id-safe and derived from the repo path.

### 7.2 Building the tree (per package)

1. **Direct nodes:** one per exporting file (or collection), holding its symbols and its primary and strong associations; path = the file's path.
2. **Indirect nodes:** one per directory config that creates a node (§8.1), holding what its `paths` list and its directory's README (primary); path = the directory.
3. **Root (Z7):** always id `root:<package dir>`, representation `'root'`, path = the package directory. A configured node exactly at a single literal scope path **merges into** the root (its `name`, `slug`, `order`, `paths`, its README), recorded as `configuredBy`.
4. **Nesting:** a node's parent is the node whose path is the nearest ancestor path of its own; nodes without one are children of the root. Collections nest by their directory. The tree is sparse.
5. **Duplicates are allowed:** a symbol listed by several nodes appears in each; modules decide what to show.
6. **Upward listings** sit at their own directory and repeat symbols; they count as check sources and are ignored when choosing link targets.

### 7.3 Issues

- Symbol issues attach to the direct node of the symbol's file, and to indirect nodes listing it when the issue concerns their resources; `verify` issues to the nodes of their file (or the nearest node above).
- Package-level analysis issues attach to the package's root; repo-level ones go to `Representation.issues` (and the repo home).
- Whether a module shows them is its choice.

### 7.4 Weak associations

An unassociated resource attaches **weakly** to the nearest node above it by path, including a node at its own directory, never to a sibling file's direct node. Weak associations exist for presentation only; this includes tests (a failing internal test stays visible). Modules may hide weak tests by default (C10).

### 7.5 Ordering

- Resources owned through `paths` follow the listed order as far as determinable (matches of one glob keep walk order); everything else follows walk order (§3.1). Children: nodes with `order` first (ascending), then walk order.
- Finer rules are deferred (§13).

### 7.6 Coverage (module concern, Z1)

Not part of the contract. The data holds what a module needs (nodes' `items`, `path`, the tree). `-base-react` defines its own notion (e.g. an indirect node covering the symbols of descendant direct nodes it lists; upward listings never cover). Per-node test summaries are likewise computed by modules from `tests` associations and `Representation.tests`.

### 7.7 Links (Z9)

The CLI decides **which node** a link targets; the URL comes from the module's `resolveRoute(target: { node: id; doc?: RepoPath; anchor?: string }, ctx: { representation })` (`anchor` = the rendered, doc-id-prefixed id). Static rules, independent of routes (D124 holds):

- **Target:** a file link is explicit; a directory link means its `README` if it has one (an implicit file link), else the directory itself.
- **Candidates:**
  - a file: first **its own nodes** (its direct node, its collection's node, configured nodes listing it, ignoring upward references), else the nodes it is **associated** with (incl. homes for home docs);
  - a directory without README: its own node (a configured node, or the root via Z7) only.
- **Selection:** strongest association first (primary > strong > weak; a file's own nodes count as primary), then shallowest tree depth, then final order.
- **No candidates:** a file → the VCS host (D73); a directory → **broken**.
- **Out of scope or outside every package** → the VCS host.
- **Broken links** (nonexistent, ignored, not represented): rendered as an `<a>` **without `href`**, with a catalogue state attribute for themes, plus `docs/broken-link` (warning by default) (revises D121).
- **Fragments:** element ids of every doc (headings, footnotes, author anchors) are **prefixed with the doc's id**; a fragment on a doc target is rewritten to the prefixed id; on other targets it is dropped. A fragment that doesn't exist in the target doc gives `docs/broken-anchor`. Authors write GitHub-style `#props` (D127, D128 parity).
- A module may answer `resolveRoute` for a node it doesn't present with another node's URL (e.g. a covering node).

### 7.8 Homes (X7, Z3)

- The repo `home` and each package's `home` hold the root's documents: **every markdown file** directly in that root, plus `README` and `LICEN[CS]E` with no extension, `.md` or `.txt` (case-insensitive; non-markdown renders as text, D71), as `DocRef`s with a **`role`** (`readme`, `licence`, `changelog`, `other`; open union).
- **The module selects** what to show (`react-default`: README and licence by default, a `homeDocs` option for more).
- **Single-package repos:** the package's home is the repo home (same id `home:.`); `layout` says which case applies; the module decides whether to merge a package's home with its root.

### 7.9 Tests in the data (X9, Z5, Z6)

```ts
interface Tests {
  status: 'ran' | 'not-run' | 'unavailable'; // open union; not-run: --no-tests or nothing required without a runner; unavailable: no runner while tests are required
  reason?: string;
  runners: {
    [id: string]: { // opaque CLI-assigned id: the display name, de-duplicated with a suffix
      name: string;
      reference?: string; // only when not a filesystem path
      status: 'completed' | 'error';
      error?: TestError;
      failedRoots?: { dir: RepoPath; error: TestError }[];
      unhandledErrors?: TestError[];
      outOfScope?: number; // listed or run files outside the in-scope tree (Z17)
      results: { [file: RepoPath]: TestFileResult[] }; // one per project; `project` kept on each
    };
  };
}
```

- Stored once; nodes reference test files through `tests: Association[]`.
- **`dev`:** one monotonic `revision` shared by data snapshots and test updates (a module applies a message only if its revision is higher). Test updates mirror the shape; merge rules: results replace per (file, project); `unhandledErrors`/`failedRoots` replace per runner when present; removed files are listed; `tests.status` is carried when it changes.

### 7.10 `Representation` and lazy docs (X7, V2)

```ts
interface Representation {
  apiVersion: string;
  revision: number; // shared with test updates (Z6)
  generatedAt: string;
  layout: 'single' | 'workspace';
  home: RepresentableNode & { info?: PackageInfo; vcs?: VcsInfo }; // id 'home:.'
  packages: { dir: RepoPath; info: PackageInfo; home: RepresentableNode; root: RepresentableNode }[];
  tests: Tests;
  issues: Issue[]; // repo-level
  meta?: JsonObject;
}
```

- The data stays plain JSON (C2 principle 2). Docs are `DocRef`s; `BuildContext`/`DevContext` offer `docs.render(ref)` (HTML, headings, assets; cached, bound to the snapshot) and `docs.read(ref)`; `DevContext.onDocsUpdate(refs)` signals re-rendered docs.
- A doc's assets are written iff `render` was called for it in this build; in `dev`, only rendered docs are re-rendered on change.
- `check` still validates every doc's links and anchors (D123), which needs parsing, not rendering.

## 8. Configuration

### 8.1 Directory configs and `directories`

- **Files:** `.tsshowcase` (JSONC; empty = `{}`) or `.tsshowcase.{json,jsonc,js,mjs,ts,mts}` (default-exported value, `defineDirectoryConfig()`), **in any in-scope directory**; several in one directory is an error. TypeScript's wildcard `include`s skip dotfiles (spike), so they stay out of `tsc` builds; `npm pack` publishes them when their directory is in `files`.
- **Value:** an object, or `true` (= `{}`); `false` is a schema error.
- **In config files:** a **`directories`** map `{ [dir]: value }` in the root and package configs (keys relative to the repo root or the package; an exception to D112 like `scope`/`ignore`). A file at the same directory **replaces** the entry key by key, wholesale, with a warning per key defined in both.
- **Creates a node** unless all its keys are non-node keys (`kinds`, `collections`, `exempt`): empty, `{}`, `true`, display-only and `paths` configs all create an indirect node; no `paths` = `paths: []` (a grouping node). New keys declare their side.
- **Keys:** `paths` (symbols by file, `file#name` or glob, and docs/tests/showcases; primary; may reach upward within scope; listing an exporting file doesn't bring its name-matched resources along), `name`, `slug`, `order`, `kinds`, `collections`, `exempt`.
- **Patterns:** picomatch relative to the config's directory, `dot: true`, globs match files, no negations in v1 (a leading `!` is rejected), `#name` split off before matching, `../` only as a literal prefix within scope; directory names with glob characters must be escaped in globs (documented); patterns the CLI builds are escaped.
- **Warnings:** a listed path that doesn't exist; a **literal** entry that isn't an exporting file, doc, test or showcase; anything pointing out of scope (X10).

### 8.2 Kind overrides

- `kinds: { '<path>#<name>': kind }`, a top-level key of the root and package configs (relative to the repo root, the package) and in directory configs and `directories` entries (relative to that directory).
- **Nearest wins:** ancestor configs of the declaring file compete first, the deepest winning (an entry and a file at the same path are merged first; a config's top-level `kinds` counts as at its base). Non-ancestor overrides count only when no ancestor sets the kind: exactly one wins; several are a check issue and no override applies. A shadowing override warns; unused keys warn.
- Values: the well-known kinds plus the module's declared kinds.

### 8.3 Collections

- A **stand-in for one exporting file**: exporting members' symbols merge into one exporting "file" named by the key, whose base drives name matching; members' own name matches are added (union); members never compete with their own collection.
- **Exporting members:** file names in the collection's directory; an exporting file belongs to one collection only (in several → error, it stands alone).
- **Resource members:** paths relative to the directory like `paths` (globs, within scope); primary; a resource may be in several collections.
- Keys may be free names; id `<dir>/<key>`; members get no direct nodes. A key clashing with a real non-member file or a directory → error, the collection isn't created.
- References to members (`paths`, `kinds`, `exempt`) keep naming the real files; the single-file rule counts a collection as one file; a missing member or one without symbols → warning, left out.

### 8.4 Exemptions (X8)

- `exempt: { '<path or glob>[#name]': ('docs' | 'tests' | 'showcase' | '*')[] }`, same places and relativity as `kinds`.
- Matching entries from every location combine (union); unused keys warn; the `check` summary reports how many symbols are exempted.

### 8.5 Other config changes

- `testRunner` (one or a list) and `testRunners` are CLI keys; `strict` (§9); `rules` accept patterns in rule ids; `outDir` stays (relative to the config file).
- Removed: `check`, `failOnTestFailure`, `units`.

## 9. Command line

### 9.1 `--strict` (X13, Z13, Z14)

| Part | Fails `build` when |
| ---- | ------------------ |
| `check` | any check error remains after severity overrides |
| `tests` | any test fails (incl. file- and suite-level errors) |
| `unavailable` | tests couldn't run although required: no runner, a runner crashed, `failedRoots` |
| `unhandled` | a runner reported `unhandledErrors` |
| `warnings` | any check warning remains (not part of a bare `--strict`) |

- `--strict` alone selects `check`, `tests`, `unavailable`, `unhandled`; parts are selected **only** in the `=` form, repeated (`--strict=tests --strict=check`). The argv normaliser turns a bare `--strict` into the default parts and handles `--no-strict` (overrides config `strict` for the run). Empty or unknown part names are rejected after parsing. Config: `strict: true | Part[]`.
- `check` honours `--strict=warnings` (non-zero exit on warnings); other parts are ignored by `check`.

### 9.2 `--rule`

`--rule <id>=<severity>` (repeatable; `error | warning | off`; split on the last `=`; an id starting with `-` uses `--rule=…`) applies on top of the root and every package's merged `rules`; rule ids may be patterns (`represent/*`), in the flag and in config alike.

### 9.3 No optional positionals

Actions never take optional positional arguments (required ones, like `init <dir>`, are fine). `build --out <dir>` (relative to the cwd) > config `outDir` (relative to the config file) > `./showcase`. With no positionals, `build --strict tests` is an unambiguous error ("`build` takes no positional arguments; did you mean `--strict=tests`?"). Migrating from an existing `showcase/` app: set `--out`/`outDir` (documented).

## 10. `init`

Re-expressed in the new model during the C8 revision (from D150): for an exporting file or collection missing resources, create the missing stubs by name (`<base>.md`, `<base>.test.*`, `<base>.showcase.*`) as its kinds require, respecting exemptions; targeting a directory or a new unit directory creates `README.md` there; a new item gets its file, stubs and the entry export. Details (targets, collections, no-entry packages) are part of the revised C8 plan.

## 11. Decision verdicts

### 11.1 C8 rounds 1–2 (D149–D159)

| Decision | Verdict |
| -------- | ------- |
| D149 showcase file format | **Changed:** several showcasing components per file, identity = export name, `shell` and titles through `ShowcasingComponent.meta` |
| D150 `init` | **Changed** (§10) |
| D151 `react/showcase-in-build` | **Kept** (a module issue through `AnalysisResult.issues`, since it reads tsconfig/`files`) |
| D152 TypeScript selection | **Kept** |
| D153 units by declaring directory and claims | **Replaced** (§7.2, §8.1) |
| D154 directory configs | **Changed:** `units` → `directories`; claims → `paths`; file replaces entry key by key; in scope only; type `DirectoryConfig` |
| D155 `showcase: false` | **Replaced** (no node; `exempt` for the check) |
| D156 `name`, `slug`, `order` | **Kept** (on nodes; slug validated at check time) |
| D157 broadened groups | **Replaced** (indirect nodes, weak associations) |
| D158 kind overrides by declaration | **Changed:** nearest wins; an error only for competing non-ancestor overrides; unused-key warning kept |
| D159 `headless` preset | **Kept** as module `verify` rules |

### 11.2 Earlier decisions touched

| Decision | Verdict |
| -------- | ------- |
| D2, D21, D40, D108 (`DirectoryInput`) | **Replaced:** CLI check + per-file `verify`; `RepoFiles` stays |
| D3, D26 (test-result mapping), D34, D42, D74, D75, D135, D136 | **Replaced:** the CLI owns runners and maps results (§4.2) |
| D5 | **Changed:** analysis = enrichments (`AnalysisResult`, §4.1) |
| D8, D76 | **Replaced** by `--strict` parts (§9.1) |
| D9, D114 | **Changed:** module `testRunners` resolve from the module's package |
| D16 | **Kept**; report over in-scope directories |
| D19 | **Re-expressed** with homes (§7.8) |
| D23 | **Changed:** lazy `docs.render`/`docs.read` |
| D29 | **Changed:** C7 before C9 (Z12) |
| D36, D37, D38 | **Replaced** (RQ1, RQ2) |
| D41 | **Changed:** dogfooding under the new rules (one-file units pass; multi-file units need collections or naming) |
| D43 | **Changed:** the CLI assembles `Representation` (not units, HTML, tests) |
| D59 | **Changed:** package configs gain `scope`, `directories`, `kinds`, `exempt` (C6 decides `strict`) |
| D61, D96, D100, D140 | **Changed:** showcases per (file, export); `shell` per component via `meta`; loaders for every file with showcasing components |
| D63 | **Changed:** required/optional methods: analysis required; `verify`, `resolveRoute`, `build`, `dev`, `scaffold`, `dispose` optional; `runTests` gone; static `testRunners`, `kinds` |
| D67 | **Changed:** routes per node via a target object |
| D68 | **Changed:** `AnalysisResult.inputs` kept; one revision shared by snapshots and test updates |
| D70 | **Changed:** validation of enrichments (symbols in existing in-scope files, known packages) |
| D72 | **Changed:** packages always present; a module may hide one |
| D73, D78, D121, D124 | **Changed:** link rules (§7.7); broken links keep a link element without `href` (D124 holds) |
| D93 | **Changed:** `build --out` (RQ7) |
| D101, D106, D107, D112 | **Changed:** package-relative `scope`; results limited to in-scope files; `directories`/`kinds`/`exempt` keys root- or package-relative |
| D116, D145, D148 | **Wording:** `testRunner` is no longer a module option |
| D120, D125 | **Changed:** assets only for rendered docs |
| D127, D128 | **Changed:** ids prefixed with the doc id |
| D141 | **Changed:** test update shape (§7.9) |
| H36 | **Replaced** by doc-id prefixing |
| U16 (08a) | **Reversed:** directory configs only in scope |

### 11.3 Untouched

Themes and isolation (C3, D81–D92, D95, D97), CLI writes (D64), doc-asset safety (D129, D130), ignore semantics (D103, D109), option marking mechanics (D116, D148), config files (D115), C15's Vite pipeline (D138, D139, D142–D147) apart from showcase loading and the data/test/doc update shapes, markdown rendering (D118, D119, D122, D126, D131) apart from link targets and id prefixing, versioning (D13, D80).

### 11.4 Open points resolved or obsolete

- C8: `testRunner` root-only text and marking (obsolete); 08a U1–U6 dissolved (U7, U8 carried to C8).
- C9: the D106 drop order and the hierarchy helper (replaced by associations and the representation builder); analysis validation (now of enrichments).
- C10: `runTests` passing `{ repo }`, `failedRoots` → unavailable units, `queued` counted as running, runner call serialisation (all CLI-side now); H36 (prefixing).
- C11: whether "tests unavailable" fails strict mode (part `unavailable`); defaulting absent units; `onAnalysisUpdate`; `LinkTargets` from the hierarchy and group routes; out-of-scope groups (C4 M6); aborting `runTests` on SIGINT (now the CLI's runners).
- L2 (relativising `TestError` paths) clearly sits with the CLI; L3 (`unhandledErrors` reach the data and `--strict`) resolved.

## 12. Effects on the existing plans

To be worked through after this revision is approved.

- **Super-plan (R10, replacing the C8 draft's unapproved R10):** the chunk texts below; edges C9 → C7 added, C10 → C7 dropped; implementation order C1 → C2 → C4 → C6 → C7 → C8 → C9, then C3 → C5 → C15 → C10 → C11 → …; the C2 CLI-duties table rewritten.
- **C1:** the unit layout (§8) and trigger-table rows; `ignore: ['/fixtures/']`, `_`-prefixed internal directories and per-unit barrels no longer needed.
- **C2 (contract):** `Representation`, `RepresentableNode`, `DocRef`, `Tests`, `Issue` placement (`Representation.issues`), `AnalyzeContext`/`AnalysisResult`, `FileSymbol`/`ShowcasingComponent` with `meta`, `SymbolKind`, per-file `verify`, `resolveRoute` target object, lazy docs on contexts (`render`, `read`, `onDocsUpdate`), test runner contract with `listTestFiles`, module statics `testRunners` and `kinds`; removed: `runTests`, `TestContext`, `loadTestRunner`, `ShowcaseData`, `UnitNode`/`GroupNode`, `AnalyzedUnit`, `unitRoot`, `RepoInput.overrides`, `ExportedSymbol.entry/typeOnly`, principle 4 ("directories are identities"); the issue model (no `dir` verdicts; `module/error` per file); inclusion rules unchanged in substance.
- **C3:** `tssData` values (test status per node, kinds), state attributes for broken links; classes or attributes for association levels and node issues as needed by C10.
- **C4:** package-relative `scope` (no package drop; per-entry unmatched warning); the typed tree; nested packages; home docs with roles; directory configs (in scope); README lookup forms.
- **C5:** link rules (§7.7); doc-id prefixing; lazy rendering; broken links as `<a>` without `href`.
- **C6:** `scope` per package; `directories`, `kinds`, `exempt`, `collections`; `testRunner`/`testRunners` as CLI keys and runner loading rewritten (candidates incl. module statics, all detecting runners); `strict`, `--strict` parts and the argv normaliser, `--rule`, rule-id patterns; `build --out`, no optional positionals; new rule ids (`represent/*`, `tests/*`) for `rules` validation; removed keys and flags; `defineDirectoryConfig`.
- **C7:** `listTestFiles`; restore around listing; the CLI drives it; Vitest 3/4 inline-project `root` documented; the "independent of cwd" test scoped.
- **C8 (React base):** rewritten: `AnalysisResult` (symbols with kinds incl. the TS rules and the syntactic fallback, showcasing components with `meta`), per-file `verify` (`headless`, showcase format), `react/showcase-in-build` as an analysis issue, scaffold; 08a group B carried over.
- **C9:** associations, the representation builder, the CLI check, the report over in-scope directories, issues → nodes, `--rule`, `check --strict=warnings`.
- **C10:** rendering `Representation` (homes with role-based selection and `homeDocs`, roots, nodes, coverage as its own notion, association levels, weak tests collapsed by default, issues, tests and per-node summaries), routes per node, showcases per (file, export), `shell` via `meta`.
- **C11:** the pipeline (§3), the CLI-run tests and their `dev` streaming with the shared revision, `--strict` parts, lazy docs and asset copying, dev invalidation, `--out`.
- **C15:** showcase loaders by (file, export); the test update shape and shared revision; lazy docs in `dev` (`onDocsUpdate`).
- **C12, C13, C14:** acceptance cases, docs (principles, conventions, the vouching effect, escaping, migration), the pack test.

## 13. Deferred

- Module-proposed collections (e.g. the `headless` preset) and glob keys in `directories` (Z10).
- Warnings for incomplete configured nodes.
- The import path (entry/subpath) of an item on its page; exported alias names.
- Finer ordering rules (children defaults, home docs).
- A module hint for preferred test runners; priority between pinned/module runners and defaults.
- Grouping issues per file in the report; a hint pointing at weakly attached resources (Z18).
- A `vouch: false` key (Z11).
