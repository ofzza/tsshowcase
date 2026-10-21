# Chunk plan C4: Repository scanning (CLI core)

> Status: **APPROVED** (round 8, 2026-10-18; reviews in `04a-c4-plan-review.md`, two rounds, fully resolved). Approved together with C2 amendment B4, C1 amendment A3 and super-plan R5, all applied. Decisions D103–D110 are in AGENTS.md §3. **C4 note (C5 planning) approved 2026-10-19 with the C5 plan:** §7 binary reads, real-path resolution and a case-exact existence check for C5 (D129, D130). **C4 amendment (C7 planning) approved 2026-10-19 with the C7 plan:** §11.1 `linkNodeModules` links every `node_modules` (nested installs, `07a-c7-plan-review.md`). **Redesign amendment (the representation redesign, `08b-representation-redesign.md`, D160–D202; plan 5 of its application) approved 2026-10-21** (choices J1–J11, D201, D202, the review's Q1–Q6 and Fix 1–20 in `04b-c4-redesign-review.md`): the seam (`ScanOptions.scope` per package, `RepoScan` with `ScannedTree`, homes and directory-config files, `inScope`/`packageOf`); package-relative `scope`, the unmatched entry as a printed warning, no package drop, the D106 paragraph removed; per-package trees stopping at nested packages, walk order files before subdirectories; `DirectoryInput` removed, tracking kept; homes with roles, README forms, the `doc` and `config` roles; directory-config files found in scope; issues, fixtures, tests, docs and recorded points; details in "Redesign amendment" below. **Backtrack from plan 6 (D205, approved by the user 2026-10-21, `05b-c5-redesign-review.md`):** §9 `vcs.ref` (`HEAD`, `CURRENT` (default, read from `.git` without git), or a name), the `.git` reader and its errors, `VcsInfo.ref`, `{ref}` in user templates; §3 the second upward-search exception; §10, §12 and the C6 recorded point. **Wording from plan 7 (C6 redesign amendment, `06b-c6-redesign-review.md`; approved 2026-10-21):** the C6 recorded point: no `include`/`exclude` migration hints (CC7); §2 `VcsOverrides` written out (keywords normalised by C6); the N16 recorded point package-relative (review); §5 `validateScope()`'s signature, `ScopeEntryError`, entries handed over as written (QI).

## Goal

Turn a repo on disk into the scan layer of the C2 input model (`RepoInput`, `RepoFiles`, `ScannedTree`; redesign): the repo and its packages, the **in-scope file tree** per package with the scan-time roles `doc` and `config` (D169), the **homes'** documents with their roles (D185), and the directory-config files found in scope (D188). Symbols and showcasing components come from the module's analysis (C8), test files from the runners (C7, listed by the CLI), associations, the representation and the check from C9, HTML from C5, config and options from C6.

C4 also ships the first fixtures and the test-only `copyFixture()` helper (C1 §5), in a new private package `-test` (D104).

## 1. Where the code lives

Everything is **internal to the CLI package** (`packages/tsshowcase/src/scan/`), not exported from its entry: plugins only see the C2 contract objects, and the check report (C9) and config (C6) are the CLI's only public types. Internal files follow the code conventions (D10) and have tests; the dogfood rules apply to exported symbols only (D161, C1 §8), so these files need no docs or showcases of their own.

```
packages/tsshowcase/src/scan/
  workspace/      detectWorkspace(): layout, package manager, package dirs (§3)
  ignore/         the "exists" verdict: hard ignores, .gitignore and .tsshowcaseignore files, config `ignore` (§4)
  scope/          scope pattern validation and matching (§5)
  walk/           the per-package walk producing the ScannedTree (§6)
  files/          RepoFiles, the read cache and read tracking (§7)
  manifest/       package.json → PackageInfo (§8)
  docs/           doc classification (roles, formats), home documents, the README preference (§8)
  configs/        directory-config file names and discovery (§8.1)
  vcs/            package.json#repository + config overrides → VcsInfo, file URL template (§9)
  scan-repo/      scanRepo(): ties it together (§2)
```

New runtime dependencies of the CLI package: `ignore` (gitignore pattern semantics), `picomatch` (scope patterns), `tinyglobby` (workspace globs), `yaml` (`pnpm-workspace.yaml`).

## 2. The seam with C6 (K9)

Two entry points, called in this order by the CLI actions (C9, C11):

```ts
/** Step 1, before any config is loaded beyond the root one: where the packages are (C6 looks for per-package config files there). */
export function detectWorkspace(root: string): Promise<WorkspaceInfo>;

export interface WorkspaceInfo {
  readonly root: string; // absolute
  readonly layout: 'single' | 'workspace';
  readonly manager?: 'npm' | 'yarn' | 'pnpm'; // set for workspaces; npm also covers bun (same `workspaces` field)
  readonly rootManifest?: RepoPath; // 'package.json' if present
  readonly packages: readonly RepoPath[]; // in walk order of the package directories (§6, 04b Fix 2); ['.'] for a single package with a manifest; [] without one
}

/**
 * Step 1b (04b Q3): whether a detected package's directory exists under §4 (ignore files, config `ignore`, hard ignores incl. the
 * output marker and `outDir`, skipped ancestors). Reads only the ignore files on the root-to-package path; C6 calls it before
 * loading a package's config, so packages that won't exist are never configured.
 */
export function packageExists(workspace: WorkspaceInfo, dir: RepoPath, options: { ignore: readonly string[]; outDir?: string; skippedPackages: readonly RepoPath[] }): Promise<boolean>;

/** Step 2, after C6 has resolved the config: the scan itself. */
export function scanRepo(options: ScanOptions): Promise<RepoScan>;

export interface ScanOptions {
  readonly workspace: WorkspaceInfo;
  readonly scope: ReadonlyMap<RepoPath, readonly string[]>; // redesign (D165): each package's effective entries, package-relative (C6 resolves flag > package config > root config); total: every existing package has an entry, a missing one throws (04b Fix 3)
  readonly ignore: readonly string[]; // D103, D107: gitignore syntax incl. `!`, relative to the root; overrides .gitignore
  readonly skippedPackages: readonly RepoPath[]; // packages with `skip: true` (D69)
  readonly outDir?: string; // absolute; only an explicitly configured or passed output dir (D101), never the implicit default
  readonly vcs?: VcsOverrides; // config `vcs` (D24), applied over detection (§9)
  readonly logger: Logger;
}

/** Config `vcs` and the `--vcs-*` flags, merged and validated by C6 (D110, D205). */
export interface VcsOverrides {
  readonly webUrl?: string;
  readonly directory?: string;
  readonly host?: 'github' | 'gitlab' | 'bitbucket';
  readonly fileUrlTemplate?: string; // contains {path}; may contain {ref}
  readonly ref?: string; // 'HEAD' | 'CURRENT' (C6 normalises the keywords to upper case, CC8) | a commit, tag or branch name
}

export interface RepoScan {
  readonly root: string;
  readonly name: string; // root manifest name, else the directory name (= Representation.home.name)
  readonly info?: PackageInfo; // root manifest
  readonly vcs?: VcsInfo;
  readonly layout: 'single' | 'workspace';
  readonly home: HomeScan; // redesign (D185): the repo root's documents (§8)
  readonly packages: readonly PackageScan[]; // in walk order of their directories (§6); every existing, non-skipped package (D193, J2)
  readonly tree: ScannedTree; // redesign (D169): C2 §5.3; scan-time roles only, 'doc' and 'config' (§6, §8)
  readonly directoryConfigs: readonly DirectoryConfigFiles[]; // redesign (D188): §8.1
  readonly files: TrackedRepoFiles; // §7
  readonly issues: readonly Issue[]; // scan issues (§10)
  inScope(path: RepoPath): boolean; // redesign (J5, 04b Q1): true iff the path is in the tree as an in-scope directory or file; false for nonexistent and ignored paths and for connectors
  packageOf(path: RepoPath): RepoPath | undefined; // redesign (J5, 04b Q1): the deepest package in `packages` whose directory contains the path, lexically (out-of-scope paths too); undefined outside every package and inside a skipped or ignored nested package (nonexistent, D103)
}

/** PackageInput minus `options`, plus its home. */
export interface PackageScan extends Omit<PackageInput, 'options'> {
  readonly home: HomeScan; // in a single-package repo, the same object as RepoScan.home
}

/** Builds the C2 object once C6 has validated each package's merged options (D69). */
export function toRepoInput(scan: RepoScan, options: ReadonlyMap<RepoPath, unknown>): RepoInput; // every package in scan.packages needs an entry; a missing one throws (04b Fix 3)
```

`inScope` and `packageOf` are pure lookups over the snapshot, with no disk access; telling a nonexistent path from an out-of-scope one (D170) stays `RepoFiles.exists`'s job (04b Q1).

The scanner never sees module options: C6 merges and validates them, and `toRepoInput` attaches them. **Scanner inputs** are `{ scope, ignore, skippedPackages, outDir }` (the super-plan's `{ include, exclude, … }`, renamed by D107; `scope` per package since the redesign), plus `vcs` (the D24 config overrides) and the logger. C4's own types are named `PackageScan`, `HomeScan`, `HomeDoc` and `DirectoryConfigFiles`, so they never clash with C2's `ScannedTree`, `ScannedPackage`, `ScannedDirectory` and `ScannedFile`, which C4 builds and uses as they are (J10).

## 3. Workspace detection

- **Repo root (L4):** the CLI passes `--cwd`, else the process cwd. **No upward search** for a config, manifest or `.git`. Positional path arguments stay relative to the process cwd (C6, C11).
- **Running inside a workspace package:** allowed, and treated as what it is, a single package. If a parent directory is a workspace root that lists this directory as a package, the CLI logs one info line ("this is package `packages/ui` of the workspace at `…`; run from there to showcase every package"). Detecting this is one of two exceptions to "no upward search" (the other: finding `.git` for `vcs.ref: CURRENT`, §9, D205) (`.gitignore` files above the root are never read, §4, M19): it reads parent `package.json` / `pnpm-workspace.yaml` files up to the filesystem root or the first `.git`, and never changes the root. (C6 adds its own error for a package-level config found at the root, L4.)
- **Managers:**
  - **pnpm:** `pnpm-workspace.yaml` at the root with a `packages` key. If present it wins (pnpm ignores `package.json#workspaces`). A `pnpm-workspace.yaml` without `packages` (pnpm 9/10 use it for settings and catalogs alone) is not a workspace declaration: detection falls through to `package.json#workspaces` (M9).
  - **npm / bun:** `package.json#workspaces` as an array.
  - **yarn:** `package.json#workspaces` as an array or `{ packages: [...] }` (yarn classic); `manager: 'yarn'` when a `yarn.lock` or `.yarnrc.yml` is present, else `npm`. The manager is informational only (logged, reported); detection is the same.
  - Lerna, Nx and Rush project lists are not read (they sit on top of a package manager's workspaces).
- **Package globs:** expanded with `tinyglobby` (directories only, `!` negations honoured, `node_modules` and `.git` never entered). A matched directory is a package only when it contains a `package.json`, as npm does, and **is not a symbolic link** (04b Q5: a symlinked package directory is skipped with an info line naming the link, consistent with M10's "symlinked directories are never walked"; documented). Ignore rules do **not** affect detection (the package manager doesn't look at them either); the scan decides visibility later (§4: a package whose directory doesn't exist isn't a package, 04b Q3). The root is never a package of its own workspace (C2 §4.3). Nested workspaces (a package declaring `workspaces`) are not followed.
- **Yarn PnP (L13):** a `.pnp.cjs` or `.pnp.loader.mjs` at the root throws: "Yarn Plug'n'Play isn't supported; set `nodeLinker: node-modules` in `.yarnrc.yml`".
- **No manifest:** `layout: 'single'`, `packages: []` (C2 §4.3), and the scan adds a `scan/no-manifest` warning. With no package nothing is in scope (D165): the tree has no packages, while the repo home's documents are still found (§8).

## 4. What exists: ignore rules (D103)

**Principle:** if it's ignored, it might as well not exist. Every other file on disk counts, whether or not git exists and whether or not the file is tracked. Nothing here calls `git`.

A path's verdict (M1, D103):

1. **Hard ignores** (never overridable, not even by `!` patterns):
   - `.git` and `node_modules`, which are also **unreachable** through `RepoFiles`, even with `ignored: true` (M21: `.git/config` may hold credentials; `node_modules` is resolved through Node, never through `RepoFiles`);
   - **output directories (D101):** the explicit `outDir`, and any directory containing tsshowcase's output marker file. The marker's **name** is defined here as a CLI constant, `.tsshowcase-output.json`; its contents and writing belong to C11. The implicit default `./showcase` is *not* ignored unless it has the marker (stdblox's `showcase/` source directory stays visible);
   - **skipped packages (D69):** every package in `skippedPackages`, with everything below it (including nested packages: a skipped package is a hard cut). A skipped package is absent from `RepoScan.packages` (C2 §10: "dropped from `RepoInput.packages`"). Combinations (04b Fix 6): a package nested below a skipped one is absent too, without a line of its own; a skipped package nested in an unskipped parent is absent and cut from the parent's tree (the parent's tree stops there as at any nested package).

**Packages that don't exist (J2, 04b Q3):** a detected package whose directory doesn't exist under this section, from any source (ignore files, config `ignore`, the output marker or `outDir`, `skip`), isn't a package: it is left out of `RepoScan.packages` and the tree, with an info line (skipped ones as before). C6 asks `packageExists()` (§2) before loading a package config, so such a package's config is never loaded or validated and its `scope` never reaches the scan; only a package's own `skip` still needs its config read.
2. Otherwise, **patterns that match the path directly decide**; a path no pattern matches directly **inherits its parent's verdict** (the root is not ignored). Two pattern sources, both in gitignore syntax:
   - **config `ignore`** (D107, renamed from `exclude`), relative to the root;
   - **`.gitignore` and `.tsshowcaseignore` files** (D109): every such file from the root down to the path's directory. A `.tsshowcaseignore` is an alias that acts **as if its lines were appended to the `.gitignore` in the same directory** (or were that directory's `.gitignore` when there is none): within a directory its lines come later and so win on conflict; across directories, deeper files win whichever kind they are; git's rules hold for the combined list, so a `!` in either can't revive a path under an ignored parent (reaching into a gitignored `dist/` takes the git idiom `!dist/`, `dist/*`, `!dist/vendor/` in a `.tsshowcaseignore` next to that `.gitignore`). It is useful on its own (an ignore list next to the content, like `.prettierignore`) and lets this repo's fixtures carry ignore rules that this repo's git doesn't apply (M2). The files themselves are listed like any other file;
   - for both kinds, with git's semantics (deeper files and later lines win; an ignore file is read if and only if its own directory is not ignored, N2). **Only `.gitignore` files at or below the root apply** (M19): none above it are read, even inside the same git repo. Running on `apps/lib` or a single workspace package therefore doesn't see the parent's `.gitignore`; that run stands on its own, and anything it should leave out goes in its own `.gitignore` or config `ignore`. Documented;
   - when config and an ignore file both match a path directly, **config wins**; among ignore files, git's order (deeper, then later) decides;
   - **one rule for all three sources (N1):** a `!` can't bring back a path whose parent is ignored, whichever source ignored the parent (git's rule). Config `ignore` only adds priority. A force-added or otherwise wanted directory inside a gitignored one is re-included with git's idiom, `ignore: ['!dist/', 'dist/*', '!dist/vendor/']`: config wins on `dist` itself, so `dist` exists and only `dist/vendor` survives below it (Q1). No path ever exists under a directory that doesn't.
3. Not applied: `.git/info/exclude` and the global `core.excludesFile` (local configuration, not `.gitignore`). Git's tracking state plays no part: a tracked file matching a pattern is ignored, an untracked file not matching one counts.

**"Matches directly" (N3)** means: among the patterns tested against this path itself (never against its ancestors), the last one that matches, in gitignore pattern semantics, with a trailing-`/` pattern matching only when the path is a directory (`read`/`exists` on an arbitrary path `lstat` it for that). Inheritance is ours, not the pattern library's: the `ignore` package's `ignores()` folds parent checks in, so C4 writes a thin matcher on the same compiled rules (or uses a per-rule test). Consequences worth documenting: `dist/**` matches everything below `dist` but not `dist` itself, so an empty `dist` stays visible. For `.gitignore`-only repos the verdict is checked against git itself: a **differential test** runs `git check-ignore` over a corpus of patterns and paths (test tooling may call git; the product doesn't).

| Patterns | Path | Verdict |
| -------- | ---- | ------- |
| `.gitignore: dist/`, config `!dist/vendor/` | `dist/vendor/a.js` | ignored (`dist` stays ignored, and a `!` can't revive below an ignored parent) |
| `.gitignore: dist/`, config `!dist/`, `dist/*`, `!dist/vendor/` | `dist/vendor/a.js` | included (config wins on `dist`; `dist/vendor` re-included, `a.js` inherits) |
| the same, plus `dist/vendor/.gitignore: *.map` | `dist/vendor/a.map` | ignored (the nested `.gitignore` matches it directly) |
| the same | `dist/other/x` | ignored (inherits from `dist/other`, ignored by `dist/*`) |
| config `legacy/` and `!legacy/keep/` | `legacy/keep/x` | ignored (the same parent rule); `legacy/*` and `!legacy/keep/` keep it |
| config `legacy/`, `legacy/.gitignore: !x` | `legacy/x` | ignored (an ignore-file `!` can't revive a path under an ignored parent) |
| `.gitignore: dist/`, `.tsshowcaseignore` (same directory): `!dist/` | `dist/x` | included (the alias's lines come later) |
| `.gitignore: dist/`, `.tsshowcaseignore` (same directory): `!dist/vendor/` | `dist/vendor/x` | ignored (git's parent rule; needs `!dist/`, `dist/*`, `!dist/vendor/`) |

The walk never enters an ignored directory: nothing below one can exist (N1).

Notes:

- Gitignore syntax applies to config patterns too, surprises included: `legacy/` ignores every `legacy` directory at any depth, `/legacy/` only the root one. Documented.
- Local clutter not covered by `.gitignore` (scratch files, `.idea/`, `coverage/`) shows up and is accepted; if it causes more than clutter, the user adds it to `.gitignore` or `ignore`. No workarounds for machine-local git configuration (Q1).
- Empty directories exist (they are on disk) and are scanned like any other. Nested clones and submodule checkouts are ordinary directories (their own `.gitignore` applies; carve them out with `ignore`).
- Symbolic links aren't an ignore rule; how the walk treats them is in §6.

**Platforms (M22):** v1 is tested on POSIX; Windows paths are normalised to POSIX separators, best effort. Pattern matching is case-sensitive (git's default; user-written patterns match as written, D201 governs only the CLI's own classifications, `04b` C), and names are sorted by UTF-16 code unit with no Unicode normalisation. Stated in the CLI README.

## 5. `scope` (D101, D107, D165)

`scope` (renamed from `include`, D107) says **where to look**: which directories are checked and showcased. It never decides what exists; ignored paths stay ignored inside it. **Since the redesign it is package-relative (D165).**

- Entries are paths or `picomatch` globs (`dot: true`, so `src/*` also matches `src/.x`) **relative to each package directory** (in a single-package repo, the repo root), POSIX separators. Absolute paths and `..` segments are rejected by C6 with a config error (C4 provides `validateScope()`). Default `['.']`.
- **The package directory and normalisation (04b Q2):** directories are matched by their package-relative path, the package directory being `.`; the entries `.` and `**` both cover it (picomatch's `**` alone wouldn't), any other entry only by matching `.` literally. `validateScope()` normalises what C6 hands over: a leading `./` removed, a trailing `/` stripped, repeated `/` collapsed, `''` and `./` → `.`; backslashes rejected with a hint to use `/` (in picomatch a backslash escapes); duplicates after normalisation merged silently. The unmatched warning names an entry as the user first wrote it. **Signature and split (C6 QI, plan 7):** `validateScope(entries: readonly string[]): readonly { written: string; normalised: string }[]`, throwing a typed `ScopeEntryError { entry, reason, hint }` for an invalid entry; C6 calls it per source (root config, each package config, `--scope`) and wraps the error into a `ConfigError` naming that source; C6 hands the entries **as written** in `ScanOptions.scope`, and `scanRepo` normalises them with the same helper, matching with `normalised` and quoting `written` (the first written form when duplicates merge). The root config sets the default for every package, a package config overrides it for its package, `--scope` applies to every package; C6 resolves that precedence and hands C4 each package's effective entries (`ScanOptions.scope`).
- **Entries match directories** (J3): an entry matching only files matches nothing (the unmatched warning below). A directory is **in scope** when it, or one of its ancestors up to the package directory, matches one of its package's entries, and it is not ignored (§4) and not at or below a nested package (§6). A file is in scope when its directory is (C2 §5.3, so `ScannedDirectory.files` is empty exactly when the directory is out of scope). Directories outside every package are never in scope; in a workspace that includes the repo root and everything outside the package directories (`fixtures/` in this repo, C1 A4).
- In-scope directories and files make up the package's `ScannedTree` (§6); their out-of-scope ancestors up to the package directory are kept in it as connectors (`inScope: false`, no files), which C9 shows as plain path segments in the report tree.
- **An entry that matches nothing** prints **one warning** (`logger.warn`, once per `scanRepo` call; repeats across `dev` rescans are C11's call; 04b Fix 5) naming the entry (redesign, J4; not a check issue, G5: it concerns the run's settings, not the repository): an entry string is unmatched when it matches no existing, in-scope directory in **any** package whose effective scope contains it; a root-config entry that every package overrides is simply unused and gets no warning; the warning names the packages it applied to. The hint depends on why (N12): an ignore file or config pattern → the git idiom in `ignore` (`!dist/`, `dist/*`, `!dist/vendor/`); a hard ignore (marker file, `outDir`, skipped package) → names it, since `!` can't revive those; a nested package → names it ("`plugins/x` is the package `packages/a/plugins/x`; give it its own `scope`"); only files matched → "`scope` entries match directories"; no match at all → `scope` entries are anchored at each package directory, so `src` matches only `<package>/src` (try `**/src`; N16). When several causes apply, the first in this order gives the hint: hard ignore, nested package, ignore pattern, files only, no match (04b Fix 5).
- **What `scope` does not limit** (D101: metadata is read wherever it lives): workspace detection, manifests, home documents (§8), the root and package config files, test configs, and every `RepoFiles` method (§7). **Directory configs are the exception: they are read only in scope** (§8.1, D169, X10).
- **No packages are dropped** (D193): a package with nothing in scope stays in `RepoScan.packages` with a tree of its package directory alone. The redesign removes the D106 drop of units and `analyze` issues outside scope: C9 limits the analysis results to in-scope files when it validates them (D170: keys outside scope dropped silently) and intersects test listings with the tree (D172), both through `RepoScan.inScope`.

## 6. The walk

- **One walk per package (redesign, J1)**, starting at the package directory, in walk order of the package directories. A package's walk **stops at nested package directories** (they are walked as packages of their own, D169) and at hard-ignored and ignored directories (§4). Directories outside every package are never walked: the ignore engine reads the `.gitignore`/`.tsshowcaseignore` files on the path from the root to each package directory lazily (in non-ignored directories only, N2), so the verdicts are the same as with a walk from the root. With no package, nothing is walked.
- Directory entries are read once (`readdir` with file types) and cached for `RepoFiles.list`.
- The walk visits every non-ignored directory of the package (globs may match at any depth) and builds the package's `ScannedPackage` (C2 §5.3): in-scope directories with their in-scope files, plus the out-of-scope ancestors connecting them to the package directory; other out-of-scope directories are pruned. Literal-prefix pruning (not descending where no entry can match) is an allowed optimisation, invisible in the result.
- **Roles at scan time** (D169, G6): `doc` (§8) and `config` (§8.1); a file without either has `roles: []`. The other roles (`exporting`, `showcase`, `test`) come from the analysis and the runners: C9 builds the every-role tree with C4's pure helper `addRoles(tree, roles: ReadonlyMap<RepoPath, readonly FileRole[]>): ScannedTree` (J11), so one module owns the tree's shape and order: it returns a new tree with the same shape and order (the scan-time tree is never changed), the given roles **unioned** with the scan-time ones, a key that isn't a file in the tree throwing (a programming error, 04b Fix 1), each file's roles de-duplicated in the fixed order `doc`, `config`, `exporting`, `test`, `showcase` (then any other role, in name order); C9 resolves test-vs-showcase exclusivity (D173) before calling it.
- **Walk order** (redesign, D169): within a directory, **files before subdirectories**, each sorted by UTF-16 code unit (`<`), never locale-dependent, recursively, so trees and reports are identical on every machine. `RepoFiles.list` keeps its plain name order (C2 §5.3).
- **Package order (04b Fix 2):** packages are ordered by the walk order of their directories (depth-first pre-order, comparing path segments in UTF-16 code units), the same in `WorkspaceInfo.packages`, `RepoScan.packages` and `ScannedTree.packages`: `packages/a`, `packages/a/plugins/x`, `packages/a-b` (a plain string sort would put `a-b` before `a/plugins/x`). A parent package therefore always precedes its nested packages, so its `skip` is known before theirs are checked (Q3).
- **Symbolic links (M10):** a symlink to a directory is listed by `list` (with `symlink: true`) but never walked (no cycles, nothing outside the root), so it is never in the tree; a symlink to a file is a file. Broken links, FIFOs, sockets and devices are skipped everywhere.
- Unreadable directories (permissions) get a `scan/unreadable` warning and are skipped; entries vanishing mid-walk are skipped silently (M20, §10). Both are tested with a mocked `fs`.
- **Unreadable package directories (04b Q4):** a package detected through its readable manifest whose directory can't be listed is kept: its manifest is read as usual (`scan/invalid-manifest` if that fails), its tree is the package directory alone (`inScope: false`, no files), its home is empty, and `scan/unreadable` names the package directory (placed on that package's root). A package directory, or a directory on the path to it, that can't be read at all is never detected (the workspace glob can't see its manifest) and raises nothing, like the package managers; documented.
- Empty directories are on disk and kept like any other (an in-scope empty directory is in the tree with no files).

## 7. `RepoFiles` and read tracking (D40, D108, K10)

**One rule for every access method (D108):** by default an ignored path doesn't exist; `{ ignored: true }` is the explicit way past that. These are the `RepoFiles` signatures of C2 §5.3 (amendment B4, below; `DirectoryInput` is gone since the redesign, D177, and `FileInput.read()` in `verify` is a tracked `read` of its file):

```ts
export interface AccessOptions {
  readonly ignored?: boolean; // true: also see ignored paths (except `.git` and `node_modules`, never reachable, M21); default false
}

read(path, options?: AccessOptions): Promise<string>; // utf-8, cached; an ignored path rejects as missing unless `ignored: true`
exists(path, options?: AccessOptions): Promise<boolean>; // false for an ignored path unless `ignored: true`
list(path?, options?: AccessOptions): Promise<DirectoryListing>; // without the option, only non-ignored entries

export interface DirectoryListing {
  readonly entries: readonly DirectoryEntry[]; // sorted by name (UTF-16 code units)
}

export interface DirectoryEntry {
  readonly name: string;
  readonly type: 'file' | 'directory'; // of the link target for symlinks; other entry kinds are never listed
  readonly symlink: boolean; // M10: symlinked directories are listed but never walked by the scan
  readonly ignored: boolean; // only ever true with `ignored: true`
  readonly reason?: 'output' | 'skipped' | 'ignore' | 'ignore-file'; // set when ignored ('ignore' = config, 'ignore-file' = .gitignore/.tsshowcaseignore, N12); the first applying rule in this order (M15); `.git`/`node_modules` are never listed (M21)
}
```

- `list` of an ignored directory without the option rejects as missing, like `read` (N14).
- `list` doesn't recurse. Listing inside an ignored directory (with `ignored: true`) is allowed, except `.git` and `node_modules`, which are unreachable through these objects in every mode (M21): `read` rejects, `exists` is false, `list` never shows them. No reinventing `fs`, just the verdict attached.
- `scope` limits none of these (§5).
- The tree (§6) describes the repo *as scanned* (in scope, never an ignored entry), which C9 hands to the module (`AnalyzeContext.tree`, `FileInput`; the requirement's "sans any ignored via `.gitignore`"); the full picture of a directory is `files.list(dir, { ignored: true })`.
- Paths are repo-relative POSIX paths. The boundary is **lexical** (M10, B4): a path that normalises outside the root is rejected; symlinks inside the root are followed (`read` of a symlinked file reads its target, as `fs` does; N7), so the boundary is a guard against mistakes, not a sandbox (C2 principle 7).
- Tools that read the disk directly (the TS compiler in `analyze`, Vitest, Vite) still see `node_modules` and build output, as after `npm install`.
- **No `isIgnored(path)`** (D108): `exists` with and without the option answers it; it can be added later, additively (D13).
- **Read tracking with `AsyncLocalStorage`:** the CLI runs each consumer inside a tracking scope (since the redesign, one per `verify` call, i.e. per file, D177); every `read`, `exists` and `list` made inside it, through **any** `RepoFiles` object (`ctx.repo.files`, `FileInput.read()`), with or without `ignored: true`, is recorded for that scope, which per-object tracking would miss. Reads of ignored files are tracked like any other, so C11 watches them. The scan itself runs outside any tracking scope: the tree is a snapshot, and C11 rescans on structure changes (adds, removes, renames).

```ts
export interface TrackedRepoFiles extends RepoFiles {
  /** Runs `fn` and returns its result together with everything it accessed. */
  track<T>(fn: () => Promise<T>): Promise<{ result: T; accessed: readonly FileAccess[] }>;
  /** dev (C11): drops cached contents and listings for these paths so the next access re-reads them. */
  invalidate(paths: readonly RepoPath[]): void; // also drops the cached listing of each path's parent, so adds and removes show (N8)
}

export interface FileAccess {
  readonly kind: 'read' | 'exists' | 'list'; // a 'read' changes on edit; 'exists' and 'list' change on add/remove
  readonly path: RepoPath;
}
```

C4 provides the mechanism; matching watcher events to recorded accesses and re-running `verify` is C11's (K10). **`readJson`:** not added (C2 §5.3).

**C4 note (C5 planning, D129, D130):** the file layer also offers (internal to the CLI) binary reads next to `read`, real-path resolution (every symlink in the path resolved) and a case-exact existence check (each segment confirmed by a `list` of its parent on case-insensitive filesystems). All three are tracked as reads like `read`/`exists`/`list` and covered by C4's tests. C5 uses them for doc assets and README links.

## 8. Manifests, documents and homes

- **`PackageInfo`** from `package.json`: `source: 'npm'`, `manifest`, `name`, `version`, `description`, `license` (as declared, also the legacy `{ type }` object form), `homepage`, `keywords` (strings only), `private`, `raw` (the parsed manifest).
- **Invalid package manifest** (unparseable JSON, not an object): the package is kept with `PackageInfo` reduced to `source` and `manifest`, plus a `scan/invalid-manifest` error on that package (`package` and `path` set). An invalid **root** manifest is fatal in either layout (M9).
- **The `doc` role (redesign, D169, C2 §5.3; J7):** a file whose name ends in `.md` (in any case, D201: `Button.md`, `Button.mD`, `Button.MD` alike); or, in any directory, `README` or `README.txt`; or, in a home's root only (the repo root and each package directory), `LICEN[CS]E`, `LICEN[CS]E.txt` and `CHANGELOG`. The names `README`, `LICEN[CS]E` and `CHANGELOG` are conventions, not names tsshowcase prescribes, and are matched case-insensitively, extension included (`readme.MD` is a doc, a markdown one; `04b` boundary). **Format** (D71): markdown for `.md` names (any case), text otherwise. **Role** (`DocRef.role`, D185): `readme`, `licence`, `changelog` for those names (in a home or not: a directory README carries `readme`), none for other docs; `other` is given to the remaining home documents (below). So `src/vendor/LICENSE` has no role and is no doc. C4 exports the classifier (`classifyDoc(name, { home })` → `{ format, role? }` or none; CLI-internal), used by C5 and C9 too. Only `.md` is markdown (`.markdown`, `.mdown` are not docs in v1; additive later).
- **Homes (redesign, D185, C2 §4.3):** the repo home and each package's home hold every non-ignored doc **directly in that root**, the root's files only:

```ts
export interface HomeScan {
  readonly dir: RepoPath; // '.' for the repo home, else the package directory
  readonly docs: readonly HomeDoc[]; // in the default home order (below, J6); the module selects what to show and may re-sort
}

export interface HomeDoc {
  readonly path: RepoPath;
  readonly format: 'markdown' | 'text';
  readonly role: 'readme' | 'licence' | 'changelog' | 'other';
}
```

  - Found whether or not the root is in scope (metadata, D101): a package's home docs exist with `scope: ['src']`; the repo root of a workspace, never in scope, has its home too. In a single-package repo the package's home is the repo home (the same object, C2 §4.3). No inheritance: a workspace package without a licence has none in its home.
  - Home documents are also files of the tree when their directory is in scope (with the `doc` role); they take no part in associations either way (D173, E1), which is C9's concern.
  - **Default order (J6, D178):** by role, `readme`, `licence`, `changelog`, `other`; within a role, markdown first, then name order (UTF-16). A module that sets no order of its own gets a sensible one, and "the first doc of a role" is a reasonable pick. C9 carries this order into the homes' `DocRef`s (`ordinal`).
  - The old single-README/single-licence lookup (an exact `README.md` winning, M15's licence precedence) is gone: every form is kept, and the module selects (D185, J6); choosing among several docs of one role is the module's rule (recorded for C10).
- **The README preference (J6):** where one README stands for a directory, among several forms the markdown ones come first, then name order (D173: a directory README's associations; D183: a directory link means its README). C4 exports `preferredReadme(fileNames)` (CLI-internal; modules can't import it), a pure function over names (README forms case-insensitive, "markdown" = an `.md` extension in any case, D201), so C5 and C9 apply the same rule; the other README files stay ordinary docs.

### 8.1 Directory-config files (redesign, D188, J8)

- **Names:** `.tsshowcase` and `.tsshowcase.{json,jsonc,js,mjs,ts,mts}`, exact case, extension included (like `tsshowcase.config.*`, D115; names tsshowcase prescribes are an exception to D201, `04b` A). `.tsshowcaseignore` is an ignore file (D109), never a directory config.
- **Found only in in-scope directories** (D169, X10): a matching file in an out-of-scope directory, a connector included, isn't read and has no role. In scope it gets the `config` role.
- **C4 finds them; it never loads them.** `RepoScan.directoryConfigs` lists each directory that has at least one, in walk order:

```ts
export interface DirectoryConfigFiles {
  readonly dir: RepoPath;
  readonly package: RepoPath;
  readonly files: readonly RepoPath[]; // name order; several = `config/several-configs` (raised by C6's loader, none of them applies)
}
```

- **The seam (J8):** C6 loads and validates them after the scan (JSONC, jiti for JS/TS, the `directory-config.schema.json`; a schema violation is a thrown `ConfigError`, D175) and raises `config/several-configs`; C9 merges every source per directory (D188: root `directories` entry < package top level < package `directories` entry < file), raises the other `config/*` issues and places them (D180). Recorded for C6 (plan 7), C9 and the super-plan's duties table (plan 10).

## 9. VCS information (D24, D110, D205)

**No git binary (N9, D110, D205).** VCS information comes from values the user wrote down: the root `package.json#repository`, overridden by config `vcs.*` keys and their flags. The CLI never runs `git` (no binary dependency, no remote or branch guessing, no inherited-environment or ownership pitfalls). Its only access to `.git` is the small reader behind `vcs.ref: CURRENT` (below, D205): `HEAD`, refs and `packed-refs`, never `config`.

| Setting | Flag | Meaning | Default |
| ------- | ---- | ------- | ------- |
| `vcs.webUrl` | `--vcs-web-url` | the repository's web page (host, owner, repo) | parsed from the root `package.json#repository` |
| `vcs.directory` | `--vcs-directory` | where the scanned root sits inside that repository (monorepo subdirectories) | `package.json#repository.directory`, else none |
| `vcs.host` | `--vcs-host` | optional: `github`, `gitlab` or `bitbucket`, for self-hosted instances (GitHub Enterprise, self-hosted GitLab, Bitbucket Server) whose URL layout is the same | detected from the host name of `webUrl` |
| `vcs.fileUrlTemplate` | `--vcs-file-url-template` | the full file-link pattern, for hosts we don't know | derived from the host of `webUrl` |
| `vcs.ref` | `--vcs-ref` | what links point at: `HEAD` (the default branch), `CURRENT` (the commit checked out, read from `.git`), or any other value as a commit, tag or branch name (D205) | `CURRENT` |

Precedence per setting: flag → config → `package.json`. The keys and flags themselves are C6's; C4 takes the resolved overrides as `ScanOptions.vcs` and produces `VcsInfo`.

- **Parsing `package.json#repository`** uses npm's own `hosted-git-info` (the same data npm uses for the "Repository" link on npmjs.com): object (`url`, `directory`) and string shorthand forms (`owner/repo`, `github:`, `gitlab:`, `bitbucket:`), URLs in `https://`, `git+https://`, `git+ssh://`, `git://`, `ssh://` and scp-like forms (M7). **Credentials in URLs are stripped** and never logged or stored. Unknown hosts: `webUrl` is the URL with `git+`, credentials and `.git` removed, host `other`.
- **Host:** `vcs.host` when set; otherwise `github.com` → github, `gitlab.com` → gitlab, `bitbucket.org` → bitbucket, anything else → `other` (no file links unless `vcs.host` or `vcs.fileUrlTemplate` is set).
- **The ref (D205, revising D110's "always `HEAD`"):** `vcs.ref` (flag → config → default `CURRENT`); the keywords `HEAD` and `CURRENT` match case-insensitively (`head`, `Current`), so a branch or tag spelt like one can't be given (documented); any other value is used verbatim (a commit, tag or branch name), URL-encoded.
  - `HEAD`: the literal `HEAD`, which GitHub, GitLab and Bitbucket resolve to the default branch.
  - `CURRENT`: the commit checked out, so links show the files as committed when the showcase was built. The `.git` reader: `.git` must be a **directory** at the scanned root or above it, found by walking up to the first `.git` (an upward search like §3's, D205); it reads `.git/HEAD`: a detached SHA is used as is; `ref: refs/heads/<branch>` is resolved through the loose ref file, else `packed-refs`. It never reads `.git/config` (credentials, M21) or any object. `.git` stays unreachable through `RepoFiles` (§4).
  - **Errors (thrown, the action stops):** `CURRENT` when `.git` is missing, is a file (worktrees, submodules), or `HEAD` or its ref can't be read or resolved: "can't read the current commit from `.git` (<why>); set `vcs.ref` (`HEAD`, or a commit, tag or branch)". No fallback to `HEAD`. **Resolved lazily (`05b` Q1):** `RepoScan.vcs` resolves the ref on first use (cached per scan) and throws only then, so it is raised only when a VCS URL is actually produced: by C5's `checkDocs` at the first link that statically goes to the VCS host (so `check`, and the check `build` and `dev` run first, fail early with this message), or by a `render` producing one through `resolveRoute` returning `undefined` (rare). A repo whose docs never link to repo files never needs `.git`. When C9 builds the representation it tries the resolution and, on failure, leaves `fileUrlTemplate` and `ref` out of the data's `VcsInfo` instead of throwing (a module then sees no VCS links available).
  - **Caveats (documented, accepted by the user):** a commit not yet pushed makes every VCS link 404 until it is; with uncommitted changes, links show the committed version while the showcase shows the working copy (D129). `build` logs the ref it used (`links pinned to 3f2a9c1`). In `dev`, the ref is read per scan.
- **`VcsInfo.fileUrlTemplate`** is fully resolved except `{path}` (relative to the scanned root), which C5 fills (URL-encoded per segment); `<ref>` is the resolved ref:
  - github: `<webUrl>/blob/<ref>/<directory/>{path}`
  - gitlab: `<webUrl>/-/blob/<ref>/<directory/>{path}`
  - bitbucket: `<webUrl>/src/<ref>/<directory/>{path}`
  - other: none, unless `vcs.fileUrlTemplate` is set.

  Directories work with the same URLs (GitHub and GitLab redirect `blob` to `tree`; Bitbucket's `src` serves both).
- **A user-supplied `vcs.fileUrlTemplate`** is used verbatim with `{path}` and the optional `{ref}` (the resolved ref, D205) as its placeholders; C4 substitutes `{ref}` (URL-encoded) before exposing `VcsInfo.fileUrlTemplate`, so `{path}` is the only placeholder C5 sees (`05b` Fix 18); `vcs.directory` doesn't apply to it (it contains any prefix itself); without `{ref}` it pins whatever it says.
- **Nothing known** (no `repository`, no config): no `VcsInfo`; repo-file links aren't turned into host URLs (C5 decides what they become) and `build` warns once (C11), naming `repository` and `vcs.webUrl`.
- `VcsInfo.kind` stays `'git'` (every supported host is a git host); `VcsInfo.defaultBranch` goes (B4); `VcsInfo.ref` holds the resolved ref (D205), so the module and C11's log can show it.

Examples (a README links to `src/Button/index.ts`; `vcs.ref: 'HEAD'` unless stated, so the examples stay stable; with the default `CURRENT`, `HEAD` becomes the checked-out commit's SHA):

| Inputs | Link |
| ------ | ---- |
| `repository: { url: 'git+https://github.com/ofzza/stdblox.git' }` | `https://github.com/ofzza/stdblox/blob/HEAD/src/Button/index.ts` |
| `repository: { url: 'https://github.com/acme/mono.git', directory: 'apps/lib' }` | `https://github.com/acme/mono/blob/HEAD/apps/lib/src/Button/index.ts` |
| the same without `directory`, plus config `vcs: { directory: 'apps/lib' }` | the same |
| config `vcs: { webUrl: 'https://gitlab.com/acme/widgets' }` | `https://gitlab.com/acme/widgets/-/blob/HEAD/src/Button/index.ts` |
| config `vcs: { webUrl: 'https://gitlab.acme.internal/acme/widgets', host: 'gitlab' }` | `https://gitlab.acme.internal/acme/widgets/-/blob/HEAD/src/Button/index.ts` |
| config `vcs: { webUrl: 'https://git.acme.internal/acme/widgets', fileUrlTemplate: 'https://git.acme.internal/acme/widgets/src/branch/main/{path}' }` | `https://git.acme.internal/acme/widgets/src/branch/main/src/Button/index.ts` |
| `repository: 'github:ofzza/stdblox'`, `vcs.ref` `CURRENT` with `.git/HEAD` detached at `3f2a9c1…` | `https://github.com/ofzza/stdblox/blob/3f2a9c1…/src/Button/index.ts` |
| the same with `--vcs-ref v1.2.0` | `https://github.com/ofzza/stdblox/blob/v1.2.0/src/Button/index.ts` |

## 10. Errors and scan issues

- **Thrown (fatal, the action stops):** root missing or not a directory; Yarn PnP; `vcs.ref: CURRENT` (the default) when the current commit can't be read from `.git` and VCS links are needed (§9, D205); an unparseable root manifest, in either layout (M9: it can't be told whether it declared a workspace); an explicit `outDir` that is the root, an ancestor of it, or contains a package (M13; C6 validates this early too); Messages name the path and say what to do.
- **Scan issues** (returned in `RepoScan.issues`, shown by `check` like any other issue, overridable through `rules`; ids and severities per D175, placement per D180, C9): `scan/no-manifest` (warning, repo-level), `scan/invalid-manifest` (error; `package` set, `path` the manifest; on the package root), `scan/unreadable` (warning; `path` is the unreadable **directory** (D202, J9); on the nearest node above it, repo-level outside every package; the directory is then skipped, M20). `scan/scope-unmatched` is no longer an issue (D175, G5): an unmatched `scope` entry prints a warning (§5).
- **Vanishing entries:** a file or directory deleted between listing and reading (`ENOENT` mid-walk, typical in `dev`) is skipped silently (M20). The rule ids are documented in the CLI README (C9 lists them with the module's rules).
- **Printed warnings:** unmatched `scope` entries (§5); a workspace that yields no package (04b Q6): one `logger.warn` per scan naming the declared globs and how many matches were left out as ignored, skipped or symlinked ("the workspace declares `packages/*` but no package was found; check `workspaces` (or `pnpm-workspace.yaml`)"); never an issue (G5).
- **Log lines only:** running inside a workspace package; a package whose directory doesn't exist under §4 (any source) is left out of `RepoScan.packages` with an info line, like a skipped one (J2, 04b Q3).

## 11. Test helpers and fixtures

### 11.1 The `-test` package and `copyFixture()` (C1 §5, D104)

A new **private workspace package `packages/tsshowcase-test/`** (`@ofzza/tsshowcase-test`), named like `-lint`: a nameless package of tooling for one activity (D27 gains the kind). It is:

- `private: true`, never published; changesets `privatePackages: { version: false, tag: false }`;
- a `devDependency` of the packages whose tests use it (the CLI first); `tsconfig.build.json` project references cover runtime dependencies only, so it never enters a build graph (M17);
- built from the C1 skeleton, so it gets its own `unit:tsshowcase-test` Vitest project, type-aware lint and the README checks with no special cases;
- skipped by this repo's dogfood `check` and showcase through its own `tsshowcase.config.json`, `{ "skip": true }` (dogfoods D59);
- the possible seed of a published testing kit for third-party module authors later (not v1).

```ts
export function fixturePath(name: string): string; // fixtures/<name> in place, for read-only tests unaffected by C1 §5's three cases

export function copyFixture(name: string, options?: CopyFixtureOptions): Promise<FixtureCopy>;

export interface CopyFixtureOptions {
  readonly linkNodeModules?: boolean; // symlink every installed node_modules of the fixture not inside another node_modules (root and packages) to the same path in the copy (default false; C7 amendment)
  readonly files?: { readonly [path: string]: string }; // extra files written relative to the copy (content a fixture can't commit, e.g. under its own .gitignore); `../x` lands in the temporary parent directory, so a test can put a `.gitignore` above the scan root (N10)
}

export interface FixtureCopy {
  readonly dir: string; // absolute; the fixture root, `<tmp>/fixture/`
  cleanup(): Promise<void>; // removes `<tmp>`
}
```

- Finds `fixtures/` from its own location (`import.meta.url`), so it works from any package's tests.
- Copies into `<tmp>/fixture/`, with `<tmp>` from `fs.mkdtemp(os.tmpdir()/tsshowcase-fixture-)`, skipping `node_modules`. No git setup: the product never runs git; since D205 it reads `.git` for `vcs.ref: CURRENT`, so a copied fixture has none, and tests needing VCS links set `vcs.ref` explicitly or write a fake `.git` (`HEAD`, a ref) through `files` (`05b` Fix 27).
- **`linkNodeModules` (C7 amendment):** links the root `node_modules` and each package's own one, so nested installs (`install-strategy=nested`, C7's `vitest-per-package`) resolve like the original. Caveats (documented): resolved paths are the original's real paths; the root `node_modules` workspace links are relative, so through the symlink they point at the original's packages; a test that edits package sources in the copy and imports them by package name needs a real root `node_modules` with fresh links (added when a test needs it).
- Own tests: copying, `node_modules` skipped, `files` (including `../`), cleanup, a nested-install copy resolving like the original.
- The N3 differential test calls `git check-ignore` (test tooling only) with every inherited `GIT_*` variable stripped and `GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_NOSYSTEM=1` (N11), so the developer's git config can't leak in; it needs a `git` binary (CI has one).

### 11.2 Fixtures added by C4

| Fixture | Purpose |
| ------- | ------- |
| `single-basic` | One package: `repository` field, `README.md`, `LICENSE` and `LICENSE.md` (markdown first within a role, Fix 10), `CHANGELOG.md`, a `.tsshowcase` at the package root (read with the default scope, not with `scope: ['src']`, Fix 10), a `CONTRIBUTING.md` (role `other`), `src/` with a few nested directories (a `README.txt`, a `src/vendor/LICENSE` with no role, a `.tsshowcase` and a directory with two directory-config files), a nested `.tsshowcaseignore` with a `!` re-include, an ignored directory with a file worth re-including through config `ignore`, a `.tsshowcase` outside `src` (not read with `scope: ['src']`), an empty directory (`.gitkeep`-free, created by the test), a symlinked directory (created by the test) |
| `workspace-npm` | `workspaces: ['packages/*', 'packages/a/plugins/*', '!packages/ignored']`, a root README and licence, package `a` (`README.md`, `LICENSE.md`, `src/`) with a nested package `packages/a/plugins/x`, package `b` (lower-case `readme.md`, a `README` too, no licence), a matched directory without `package.json`, a package ignored by `.tsshowcaseignore` |
| `workspace-yarn` | `workspaces: { packages: [...] }` plus `yarn.lock` |
| `workspace-pnpm` | `pnpm-workspace.yaml`, plus a `package.json#workspaces` it overrides |
| `no-manifest` | A README and some directories, no `package.json` |
| `broken-manifest` | An unparseable package manifest in a workspace package |
| `broken-pnp` | A `.pnp.cjs` at the root |

- **Prettier and intentionally broken files (01a):** fixtures whose name starts with `broken-` are excluded from Prettier as a whole (`.prettierignore`: `fixtures/broken-*/`). Broken fixtures contain only what the broken case needs. ESLint already ignores `fixtures/**` (C1).
- **Fixtures carry their ignore rules in `.tsshowcaseignore` files** (D109, M2), so this repo's git commits everything they contain. Tests of `.gitignore` itself (the file is found, git's parent rule) are crafted so that the fixture's `.gitignore` never keeps a needed file out of this repo's commits: they ignore only content the test creates (`copyFixture`'s `files` option) or that the test expects to be absent. Content this repo's own `.gitignore` would catch (a fixture `dist/`) is re-included with `!` rules at the root, in `fixtures/.gitignore` or in the fixture (C1 amendment A3).
- A parent `.gitignore` that must not apply (M19) is written above the copy with `files: { '../.gitignore': … }`.
- `node_modules` directories, empty directories, symlinks and the marker file are created at test time.
- The fixtures README (C1) gets the table above.

## 12. Tests

Per directory of §1, plus `scanRepo` end to end over every fixture:

- the verdict (§4): hard ignores (`.git`, `node_modules`, explicit `outDir`, the marker file and the default `showcase/` staying visible without it, skipped packages with nested packages) not overridable by `!`; nested `.gitignore`, `.gitignore` re-includes and their git limit; `.tsshowcaseignore` alone, next to a `.gitignore` (its lines winning), deeper vs shallower across kinds, the git idiom for reaching into a gitignored directory; no `.gitignore` above the root applied, even inside the same git repo (M19); `.git/info/exclude` having no effect; a tracked file matching `.gitignore` being ignored; config `ignore` adding patterns and the git idiom re-including into a gitignored directory, every row of the §4 verdict table (direct match, inheritance, config over `.gitignore`, no `!` reviving below an ignored parent from any source) (M1, N1); direct-match edge cases: `dist/**`, `dist/*`, a trailing-`/` pattern against a file of that name, slash-less negations; ignore files read only in non-ignored directories (N2); the differential test against `git check-ignore` (N3); the walk never entering ignored directories; empty directories; a nested clone being scanned; symlinks;
- scope: default, plain paths and globs relative to each package (incl. dot directories), the package directory covered by `.` and `**` (04b Q2), every normalisation rule and the backslash rejection, a package's own scope replacing the default, connectors kept with `inScope: false` and no files, out-of-scope branches pruned, entries matching only files, directories outside packages never in scope (the workspace root, `fixtures/`), `outDir` validation, the unmatched warning (once per entry across packages, no warning for an overridden default, every N12 hint incl. the nested-package one), packages with nothing in scope kept, `inScope` and `packageOf` for files and directories (nonexistent, ignored, connector and out-of-scope paths; paths inside skipped or ignored nested packages → `undefined`, 04b Q1), validation errors;
- the tree: `addRoles` unioning roles and throwing on a key that isn't a file in the tree (Fix 1); package order with `packages/a`, `packages/a-b`, `packages/a/plugins/x` (Fix 2); a skipped package nested in an unskipped parent (Fix 6); unreadable package directories, listable-manifest and fully unreadable (mocked `fs`, 04b Q4); per-package trees stopping at nested packages (and the nested package's own tree), walk order (files before subdirectories), symlinked directories absent, empty directories kept, scan-time roles only (`doc`, `config`, else `[]`), `addRoles` keeping shape and order, a skipped package's subtree absent, a package ignored by an ignore file, by config `ignore` or by holding the output marker left out with an info line, `packageExists()` agreeing with the scan for each (04b Q3);
- docs and homes: the `doc` role and format for every accepted name and casing (`classifyDoc` unit tests, not fixture files, since names differing only in case collide on case-insensitive filesystems, Fix 7: `Button.mD`, `Button.MD`, `readme.MD`, `README`, `README.txt`, `License.TXT`, `CHANGELOG`), `LICENSE`/`CHANGELOG` without a role outside home roots, home docs in the default home order (J6, Fix 14; `LICENSE.md` before `LICENSE`) for the repo and each package, a `.tsshowcase` at the package root read with the default scope and not with `['src']` (Fix 10), home docs of out-of-scope roots, the single-package home shared, `preferredReadme`;
- directory configs: every accepted name, exact case incl. the extension (`.tsshowcase.JSON` is no directory config), `.tsshowcaseignore` not one, found in scope only (not in connectors or out-of-scope directories), several in one directory listed together, the `config` role;
- workspace detection: a workspace yielding no package (the printed warning, 04b Q6), each manager, negations, directories without manifests, a symlinked package directory skipped with an info line (created at test time, 04b Q5), pnpm precedence and a settings-only `pnpm-workspace.yaml`, PnP error, unparseable root manifests, inside-a-workspace-package info;
- `RepoFiles`: `read`/`exists`/`list` with and without `ignored: true` (entries, `symlink`, reasons and their priority), caching, root escape rejected, relative and parent paths, parent listings invalidated (N8), `list` of an ignored directory rejecting (N14), `track()` attribution under concurrent scopes (two interleaved per-file `verify`-like calls, including reads with `ignored: true`), `invalidate`;
- manifests, an invalid package manifest (the issue's `package` and `path`), single-package collapse;
- VCS: every `repository` form and host (incl. `git+https`, `git+ssh`, `git://`, shorthand), credential stripping, `directory` from `package.json` and from config, config/flag precedence, `vcs.host` on a self-hosted URL, a user template used verbatim (with and without `{ref}`), nothing known → no `VcsInfo`, every row of the §9 examples; `vcs.ref`: `HEAD`/`CURRENT` in any case, an explicit value, `CURRENT` with a detached `HEAD`, a branch ref (loose and in `packed-refs`), `.git` above the root, and every error case (no `.git`, `.git` a file, unreadable or unresolvable `HEAD`), no error when no VCS links are needed (a fake `.git` written at test time; `.git/config` never read, asserted); no `git` process is ever spawned (asserted);
- determinism: the same fixture scanned twice gives deeply equal results, and the order doesn't depend on locale.

## 13. Documentation

- CLI README: how the repo is scanned (root, what exists: hard ignores, `.gitignore` and `.tsshowcaseignore`, config `ignore` and its priority (with the git idiom for re-including inside an ignored directory), the gitignore-syntax surprise; `scope`, package-relative and matching directories, with `scope: ['src']` recommended for typical layouts and the root of a workspace never in scope; workspaces and nested packages; walk order; which files are docs and home documents, with their roles; the directory-config file names and that they are read only in scope; the marker file; VCS settings (`package.json#repository`, `vcs.*`, always `HEAD`, no git); Yarn PnP), and the `scan/*` rule ids. The config keys themselves are documented with C6.
- `-api` docs for `RepoFiles` (B4): the "ignored doesn't exist unless asked" rule and why consumers that discover files should not pass `ignored: true` casually; the `ScannedTree` (B9) with its stage-dependent roles is documented with C2.
- `-test` README; CONTRIBUTE.md: `copyFixture`, the `broken-*` fixture naming, fixtures using `.tsshowcaseignore` and `!` re-includes against this repo's `.gitignore`. The C1 §5 gotcha text points at the helper.
- AGENTS.md trigger table: a new fixture row ("new fixture → fixtures README table").

## Acceptance criteria

- `detectWorkspace` and `scanRepo` produce the documented results for every fixture, deterministically.
- `toRepoInput` output satisfies the C2 `RepoInput` type, and `RepoScan.tree` the C2 `ScannedTree` type (as amended by B4 and B9; type tests).
- Read tracking attributes accesses to the right scope under concurrency.
- No credential from a `repository` URL appears in `VcsInfo`, logs or errors.
- `npm run ci` passes, including Prettier with the `broken-*` fixtures and the `-test` package's own tests.

## Out of scope

Config loading, per-package config files, loading and validating directory configs, `scope`/`ignore` key and flag parsing (C6); merging directory configs, limiting analysis results and test listings to in-scope files, associations, the representation, the check and the report (C9); matching watcher events to tracked accesses, rescans, the marker file's contents and writing (C11); filling `{path}` and rendering links (C5); analysis (C8); listing test files (C7, driven by the CLI).

## Recorded for later chunks

- **C5:** `{path}` is the only placeholder C5 fills (*D205: C4 substitutes `{ref}` first*), URL-encoded per segment; no `fileUrlTemplate` (no VCS info, or host `other` without a template) → decide what a repo-file link becomes. ~~Links to dropped units (out of scope) and to ignored files need an explicit mapping (M6).~~ *Redesign:* settled by D183 (ignored → broken, out of scope → the VCS host), using `RepoScan.inScope`; directory links use `preferredReadme`, docs `classifyDoc` (§8).
- **C6:** calls `detectWorkspace` first, then loads per-package configs from `WorkspaceInfo.packages`, skipping packages for which `packageExists()` is false (04b Q3: their configs are never loaded), then `scanRepo` (redesign: with each package's effective `scope`, flag > package config > root config, `--scope` replacing both; `validateScope()` package-relative, normalising entries, 04b Q2), then loads the directory-config files in `RepoScan.directoryConfigs` (§8.1: validation, `config/several-configs`); `scan/scope-unmatched` is no rule id any more (D175); config keys `scope` and `ignore` with repeatable `--scope` / `--ignore` flags (D107); flags vs config arrays and their base directory (M12, recommendation: `--ignore` appends, `--scope` replaces; `--ignore` relative to the repo root, `--scope` package-relative since the redesign, D165); an unknown `include` or `exclude` key is an ordinary unknown key (no migration hint: never published, C6 CC7, plan 7); validates `scope` with `validateScope()`; validates an explicit `outDir` early (M13); error for a package-level config at the root (L4); `scan/*` and `module/error` are CLI-known rule ids for `rules` validation (M15); attaches merged options with `toRepoInput`.
- **C10 (J6):** choosing among several home docs of one role (e.g. `LICENSE` and `LICENSE.md`) is the module's rule; the data arrives in the default home order (§8), so "the first of the role" is a sound default.
- **C7, C10:** discovery through `RepoFiles.list` without `ignored: true`; `DirectoryListing` is `entries` (B4); tests in out-of-scope directories still run and ~~end up unmapped (M6)~~ are counted per runner as `outOfScope` (redesign, D172, D186).
- **C8:** module-side memoisation (a promise cached in `state` during `analyze` and awaited in `verify`) defeats `AsyncLocalStorage` read tracking; document it for module authors (C13) and avoid it in `-base-react` (M11). Fixtures with a committed `dist/` re-include it against this repo's `.gitignore` with `!` rules (C1 amendment A3, D109).
- **C9:** report tree with out-of-scope connectors (`inScope: false`) as plain path segments; `scan/*` issues and their placement (D180; `scan/unreadable`'s `path` is a directory, J9, D202); ~~the M5 order for dropping out-of-scope units (D106)~~ and ~~`analyze` issues whose `dir` isn't scanned (N17)~~ replaced by the redesign: analysis keys outside scope are dropped silently (D170) through `RepoScan.inScope`, and test listings intersected with the tree (D172); the every-role tree through `addRoles` (§6, J11); merging directory configs and the remaining `config/*` issues (§8.1, J8); ~~this repo's config `ignore: ['/fixtures/']` (anchored, N13)~~ (dropped by C1 A4: `fixtures/` lies outside every package, so it is never in scope; the `-test` package skips itself).
- **C11:** the marker file `.tsshowcase-output.json` (contents, writing; no build-machine paths in it, since it is deployed); `track()` results drive dev re-verify (per file, D177), including reads of ignored files; `invalidate()` on watcher events; a change to any `.gitignore` or `.tsshowcaseignore` at or below the root, the marker file or the config changes the verdict for whole subtrees → a full rescan and a new `RepoScan` (config → restart) (M11); redesign: the tree is a snapshot, so adds, removes and renames in a package, and any change to a directory-config file, mean a rescan too (§7); ~~groups outside scope in the hierarchy and D78 group READMEs (M6)~~ (obsolete: no groups, D179).
- **C14:** `-test` is private; the publish script skips it.
- **C5 (tests):** confirm `HEAD` links for every host, Bitbucket's `src/HEAD/…` included *(settled by D205: C5 tests every ref form)*.
- **C6 (D110, D205):** `vcs.webUrl`, `vcs.directory`, optional `vcs.host`, `vcs.fileUrlTemplate`, `vcs.ref` (D205: `HEAD`, `CURRENT` or a name; default `CURRENT`) keys with `--vcs-web-url`, `--vcs-directory`, `--vcs-host`, `--vcs-file-url-template`, `--vcs-ref`; `vcs.defaultBranch` removed from the schema.
- **C11:** the one-time `build` warning when READMEs link to repo files and no VCS info is known.
- **C6/C13 docs (N16):** `scope` uses package-relative paths and globs (D165), `ignore` root-relative gitignore syntax (unanchored without a slash); the asymmetry is documented next to both keys.

## Redesign amendment (plan 5 of the representation redesign's application; approved 2026-10-21)

Plan 5 of the redesign's application (`08b-representation-redesign.md` §12, `08e` E7), after AGENTS (plan 1), C2 B9/B10, C1 A4 and C3-A3; the choices J1–J11 and the review are in `04b-c4-redesign-review.md`. C4 now produces what the redesigned contract's scan layer needs (D165, D169, D185, D188) instead of the directory list `verify` used to receive:

- **Goal, §1:** the scan layer of C2 (`RepoInput`, `RepoFiles`, `ScannedTree`), homes and directory-config files; `scan/docs/` classifies docs, `scan/configs/` finds directory configs; "not units (D41 …)" → the dogfood rules apply to exported symbols only (C1 §8; `01e` stale wording).
- **§2 seam:** `ScanOptions.scope` per package; `RepoScan` loses `readme`, `licence`, `directories` and `isScanned`, gains `home`, `tree`, `directoryConfigs`, `inScope`, `packageOf` (J5); packages as `PackageScan` with their `home`; C4's own types renamed so they don't clash with C2's (J10; 02b's C4 checklist).
- **§3, §4:** no manifest → nothing in scope, the repo home still found; skipped packages absent from `packages`.
- **§5 `scope`:** package-relative, matching directories only (J3); in scope = C2 §5.3's definition; connectors in the tree; the unmatched entry is a printed warning once per entry across packages, with a nested-package and a files-only hint (J4, G5); directory configs the one metadata read only in scope; no package drop (D193); the D106 paragraph removed (D170, D172).
- **§6 walk:** one walk per package, stopping at nested packages, nothing outside packages walked (J1); scan-time roles `doc` and `config`, the every-role tree through `addRoles` (J11); files before subdirectories (D169); symlinked directories absent from the tree.
- **§7:** `DirectoryInput` and `TrackedRepoFiles.directory()` removed (D177); tracking per `verify` call (per file); the scan is a snapshot.
- **§8:** the `doc` role, formats and roles (J7), homes with every root doc in the default home order (by role, then markdown first, then name; D185, D178, J6), M15's single-licence and README-casing preference replaced by the module's selection plus the CLI-internal `preferredReadme` for directory READMEs and links (J6); **§8.1** directory-config files: names, in scope only, found by C4, loaded by C6, merged by C9 (J8).
- **§10:** `scan/scope-unmatched` → printed warning; `scan/unreadable` carries the directory in `path` (J9; `Issue.file` renamed `path`, D202); a package ignored by an ignore file isn't a package (J2); dropped-unit log lines gone.
- **§11.2, §12, §13, acceptance, out of scope:** fixtures extended (home docs with roles, directory configs, a nested package); tests for the tree, homes, docs, directory configs and package-relative scope, the dropped-unit tests removed; the `DirectoryInput` type test → `ScannedTree`; docs for the new behaviour.
- **Recorded for later chunks:** C5's dropped-unit mapping (settled by D183), C9's M5 order and N17 drop (replaced by D170), C11's groups (obsolete), "unmapped" → `outOfScope`; new points for C6 (effective `scope`, the directory-config loader), C9 (`addRoles`, merging, placement), C11 (rescans).
- **Review outcomes (`04b` "Agent review"):** Q1 `inScope`/`packageOf` semantics; Q2 the package directory as `.` (covered by `.` and `**`) and entry normalisation; Q3 `packageExists()`, packages that don't exist from any §4 source, their configs never loaded; Q4 unreadable package directories; Q5 symlinked package directories aren't packages; Q6 a printed warning for a workspace with no package; Fix 1–20 (precision, package order, tests and fixtures, history markers, wording in C2, C5, 08b, AGENTS).
- **Also decided in plan 5:** D201 (extensions compared case-insensitively, with exceptions for tsshowcase's own file names, tools' own names and user-written patterns and paths) and D202 (`Issue.file` → `Issue.path`), applied across the plans.
- **Unchanged:** what exists (§4, D103, D108, D109) apart from Q3's "packages that don't exist", workspace detection (§3) apart from Q5 and Q6, `RepoFiles` and its access rule, VCS (§9), the `-test` package (§11.1).

## C2 amendment B4 (re-approved with this plan, applied)

> History: superseded where the redesign amendment says so (`DirectoryInput` removed, D177; C2's scan types per B9).

- **§5.3 `RepoFiles`:** `read`/`exists`/`list` take `options?: { ignored?: boolean }`; by default an ignored path doesn't exist (read rejects as missing, `exists` is false, `list` leaves it out), `ignored: true` sees ignored paths too, except `.git` and `node_modules`, which are never reachable (D108, M21). `DirectoryListing` becomes `{ entries: DirectoryEntry[] }` with `DirectoryEntry { name, type, symlink, ignored, reason? }`. The comment "non-ignored entries only" becomes the default-behaviour note, and the JSDoc's root-escape sentence becomes "lexical boundary, symlinks followed" (N6).
- **§5.5 `DirectoryInput`:** the same three signatures; `files`/`directories` stay filtered names (the directory as scanned). The scan sentence: `.gitignore` and `.tsshowcaseignore` files at or below the root (not above it, not `.git/info/exclude`), config `ignore` patterns (overriding `.gitignore`, `!` included), directories outside `scope`; `exclude`/`include` renamed (D107).
- **§4 `VcsInfo`:** `defaultBranch` removed; `fileUrlTemplate` documented as fully resolved except `{path}` (D110).
- **§10 config sketch:** `vcs` becomes `{ webUrl?, directory?, host?, fileUrlTemplate? }`, `defaultBranch` removed (D110); `include` → `scope` ("where to look: the directories checked and showcased; never un-ignores"), `exclude` → `ignore` ("gitignore syntax incl. `!`; overrides `.gitignore`").
- **Also stale in C2 (M3):** the §5.3 comment block under `RepoFiles` ("the gitignore-aware view … plain `fs` would descend into ignored directories") rewritten for the default/opt-in rule, and `list(dir)` takes an optional path; §5.4's D72 paragraph and §5.5's opening ("every non-ignored directory") become "every scanned directory (in `scope`, not ignored)"; §5.5's "the repo root (`'.'`) is verified too" becomes "when it is in scope"; the root-escape sentences become lexical with symlinks followed (M10); the §4 `VcsInfo.fileUrlTemplate` example becomes `'https://github.com/o/r/blob/HEAD/{path}'` with a note that `{path}` is the only placeholder (D110).

## Super-plan R5 and AGENTS edits (applied)

> History: superseded where the redesign amendment says so (D106's drop replaced by D165/D170; super-plan R10 follows in plan 10).

- **R5 (C4 planning, D103–D110):** C4's "Git info: remote → host, default branch … fallback" bullet → VCS from `package.json#repository` and config only, no git (D110); C4 bullets: `.gitignore` files at or below the root only, plus the `.tsshowcaseignore` alias (no `.git/info/exclude`), config `ignore` overrides with `!`, hard ignores, `scope`, scanner inputs `{ scope, ignore, skippedPackages, outDir }`, the `-test` package. C6: `scope`/`ignore` keys and flags. C8: "units outside `include` are dropped" moves to the CLI (C9, D106). C9: the drop in the duties table, and "tree report of every non-ignored directory" → "every scanned directory" (N15). Package table and C1's package list: `-test` (private). D27's kinds gain `test`.
- **C1 amendment A3 (M4):** §1 layout and §2 package and dependency tables gain the private `-test` package (devDependency only, no project reference); §6 changesets `privatePackages: { version: false, tag: false }`; §5 "C4 also decides where the helper lives" → "lives in `-test` (C4)"; "Recorded for later chunks" C9: `exclude` → `ignore`; §5 "Prettier skips only their `node_modules`" → also `fixtures/broken-*/`; §5 "must be excluded via this repo's tsshowcase config" → "ignored via config `ignore`"; §5 gotcha 2 notes that the root `.gitignore` no longer leaks into fixture scans (M19), while git state still does (N15); §5 states that fixtures needing content this repo's `.gitignore` would catch re-include it with `!` rules (at the root, in `fixtures/.gitignore` or per fixture), and that fixtures use `.tsshowcaseignore` for their own ignore rules (M2, D109).
- **AGENTS:** D103–D110 added (D110 revises D24 and D105); D101's text updated for the renames and for `ignore` deciding what exists, its status "revised by D103, D106, D107"; §1 summary ("all directories not ignored by `.gitignore`") and §1's "receives every non-ignored subdirectory" note the config `ignore` and `scope` (N15); D27, D14 mention `-test`; D41 notes `-test`'s `skip`; open points: C4 removed, C6/C8/C9 rows updated (`scope`, `ignore`, D106, `fixtures/` via `ignore`), plus the M6/M11/M12 points; C4 marked approved in the tracker; §6 pitfalls unchanged.

## Open questions

- ✅ **Q1 What exists (D103, D107, D108, D109):** `.gitignore` files decide, regardless of git and of tracking (no `.git/info/exclude`, no global excludes, only `.gitignore` files at or below the root, M19); config `ignore` (renamed from `exclude`) overrides them on the same path, `!` included, under git's parent rule like every source (N1); hard ignores stay. A `.tsshowcaseignore` alias acts as lines appended to the same directory's `.gitignore` (D109, from M2). `scope` (renamed from `include`) only says where to look. Every access method treats ignored paths as nonexistent unless called with `{ ignored: true }`; `list` returns `DirectoryEntry` descriptors. (Discussed over several rounds: Claude first recommended JS rules with `.git/info/exclude`, then a git-backed "fresh clone" model; the user settled on `.gitignore` only, a separate `ignore` override and the rename, and asked for one consistent access rule.)
- ✅ **Q2 `copyFixture` home (D104):** a private `-test` package (the user's choice after weighing a root `test-utils/` directory; named like `-lint`).
- ✅ **Q3 Branch fallback (D105, revised by D110):** `origin/HEAD`, else the literal `HEAD` in URLs; since N9, always `HEAD` (no git).
- ✅ **Q4 Units outside `scope` (D106)** *(history: replaced by the redesign, D165, D170; see §5)*: dropped by the CLI after `analyze`, in the M5 order (existence check, drop, remaining D70 checks); the C8 open point changes accordingly.
