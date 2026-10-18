# Chunk plan C4: Repository scanning (CLI core)

> Status: **APPROVED** (round 8, 2026-10-18; reviews in `04a-c4-plan-review.md`, two rounds, fully resolved). Approved together with C2 amendment B4, C1 amendment A3 and super-plan R5, all applied. Decisions D103–D110 are in AGENTS.md §3.

## Goal

Turn a repo on disk into the repo / package / directory layer of the C2 input model (`RepoInput`, `RepoFiles`, `DirectoryInput`), plus the list of directories that `check` visits. Units come from the module's `analyze` (C8), tests from the runner (C7), HTML from C5, config and options from C6.

C4 also ships the first fixtures and the test-only `copyFixture()` helper (C1 §5), in a new private package `-test` (D104).

## 1. Where the code lives

Everything is **internal to the CLI package** (`packages/tsshowcase/src/scan/`), not exported from its entry: plugins only see the C2 contract objects, and the check report (C9) and config (C6) are the CLI's only public types. Internal directories follow the code conventions (D10) and have tests, but are not units (D41 applies to exported units).

```
packages/tsshowcase/src/scan/
  workspace/      detectWorkspace(): layout, package manager, package dirs (§3)
  ignore/         the "exists" verdict: hard ignores, .gitignore and .tsshowcaseignore files, config `ignore` (§4)
  scope/          scope pattern validation and matching (§5)
  walk/           the directory walk producing ScannedDirectory[] (§6)
  files/          RepoFiles, DirectoryInput, the read cache and read tracking (§7)
  manifest/       package.json → PackageInfo (§8)
  docs/           README and licence lookup (§8)
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
  readonly packages: readonly RepoPath[]; // sorted; ['.'] for a single package with a manifest; [] without one
}

/** Step 2, after C6 has resolved the config: the scan itself. */
export function scanRepo(options: ScanOptions): Promise<RepoScan>;

export interface ScanOptions {
  readonly workspace: WorkspaceInfo;
  readonly scope: readonly string[]; // D101, D107; default ['.']
  readonly ignore: readonly string[]; // D103, D107: gitignore syntax incl. `!`, relative to the root; overrides .gitignore
  readonly skippedPackages: readonly RepoPath[]; // packages with `skip: true` (D69)
  readonly outDir?: string; // absolute; only an explicitly configured or passed output dir (D101), never the implicit default
  readonly vcs?: VcsOverrides; // config `vcs` (D24), applied over detection (§9)
  readonly logger: Logger;
}

export interface RepoScan {
  readonly root: string;
  readonly name: string; // root manifest name, else the directory name
  readonly info?: PackageInfo; // root manifest
  readonly readme?: RepoPath; // root README (for RepoNode, C11)
  readonly licence?: RepoPath; // root licence
  readonly vcs?: VcsInfo;
  readonly layout: 'single' | 'workspace';
  readonly packages: readonly ScannedPackage[]; // PackageInput minus `options`
  readonly directories: readonly ScannedDirectory[]; // every directory check visits, in walk order (§6)
  readonly files: TrackedRepoFiles; // §7
  readonly issues: readonly Issue[]; // scan issues (§10)
  isScanned(dir: RepoPath): boolean; // D101: in `scope` and not ignored, i.e. scanned (§5)
}

export type ScannedPackage = Omit<PackageInput, 'options'>;

/** Builds the C2 object once C6 has validated each package's merged options (D69). */
export function toRepoInput(scan: RepoScan, options: ReadonlyMap<RepoPath, unknown>): RepoInput;
```

The scanner never sees module options: C6 merges and validates them, and `toRepoInput` attaches them. **Scanner inputs** are `{ scope, ignore, skippedPackages, outDir }` (the super-plan's `{ include, exclude, … }`, renamed by D107), plus `vcs` (the D24 config overrides) and the logger.

## 3. Workspace detection

- **Repo root (L4):** the CLI passes `--cwd`, else the process cwd. **No upward search** for a config, manifest or `.git`. Positional path arguments stay relative to the process cwd (C6, C11).
- **Running inside a workspace package:** allowed, and treated as what it is, a single package. If a parent directory is a workspace root that lists this directory as a package, the CLI logs one info line ("this is package `packages/ui` of the workspace at `…`; run from there to showcase every package"). Detecting this is the only exception to "no upward search" (`.gitignore` files above the root are never read, §4, M19): it reads parent `package.json` / `pnpm-workspace.yaml` files up to the filesystem root or the first `.git`, and never changes the root. (C6 adds its own error for a package-level config found at the root, L4.)
- **Managers:**
  - **pnpm:** `pnpm-workspace.yaml` at the root with a `packages` key. If present it wins (pnpm ignores `package.json#workspaces`). A `pnpm-workspace.yaml` without `packages` (pnpm 9/10 use it for settings and catalogs alone) is not a workspace declaration: detection falls through to `package.json#workspaces` (M9).
  - **npm / bun:** `package.json#workspaces` as an array.
  - **yarn:** `package.json#workspaces` as an array or `{ packages: [...] }` (yarn classic); `manager: 'yarn'` when a `yarn.lock` or `.yarnrc.yml` is present, else `npm`. The manager is informational only (logged, reported); detection is the same.
  - Lerna, Nx and Rush project lists are not read (they sit on top of a package manager's workspaces).
- **Package globs:** expanded with `tinyglobby` (directories only, `!` negations honoured, `node_modules` and `.git` never entered). A matched directory is a package only when it contains a `package.json`, as npm does. Ignore rules do **not** affect detection (the package manager doesn't look at them either); the scan decides visibility later (§5). The root is never a package of its own workspace (C2 §4.3). Nested workspaces (a package declaring `workspaces`) are not followed.
- **Yarn PnP (L13):** a `.pnp.cjs` or `.pnp.loader.mjs` at the root throws: "Yarn Plug'n'Play isn't supported; set `nodeLinker: node-modules` in `.yarnrc.yml`".
- **No manifest:** `layout: 'single'`, `packages: []` (C2 §4.3), and the scan adds a `scan/no-manifest` warning.

## 4. What exists: ignore rules (D103)

**Principle:** if it's ignored, it might as well not exist. Every other file on disk counts, whether or not git exists and whether or not the file is tracked. Nothing here calls `git`.

A path's verdict (M1, D103):

1. **Hard ignores** (never overridable, not even by `!` patterns):
   - `.git` and `node_modules`, which are also **unreachable** through `RepoFiles`/`DirectoryInput`, even with `ignored: true` (M21: `.git/config` may hold credentials; `node_modules` is resolved through Node, never through `RepoFiles`);
   - **output directories (D101):** the explicit `outDir`, and any directory containing tsshowcase's output marker file. The marker's **name** is defined here as a CLI constant, `.tsshowcase-output.json`; its contents and writing belong to C11. The implicit default `./showcase` is *not* ignored unless it has the marker (stdblox's `showcase/` source directory stays visible);
   - **skipped packages (D69):** every package in `skippedPackages`, with everything below it (including nested packages: a skipped package is a hard cut).
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

**Platforms (M22):** v1 is tested on POSIX; Windows paths are normalised to POSIX separators, best effort. Pattern matching is case-sensitive (git's default), and names are sorted by UTF-16 code unit with no Unicode normalisation. Stated in the CLI README.

## 5. `scope` (D101, D107)

`scope` (renamed from `include`, D107) says **where to look**: which directories are checked and showcased. It never decides what exists; ignored paths stay ignored inside it.

- Entries are paths or `picomatch` globs (`dot: true`, so `packages/*` also matches `packages/.x`) relative to the repo root, POSIX separators. Absolute paths and `..` segments are rejected by C6 with a config error (C4 provides `validateScope()`); an entry matching only files simply matches no directory (the unmatched warning below). Default `['.']`.
- A directory is **in scope** when it, or one of its ancestors, matches an entry. Only directories in scope and not ignored (§4) are **scanned**: they appear in `RepoScan.directories`, get `verify`, appear in the report.
- Their ancestors are walked through (to reach them) but not scanned; C9 shows them as plain path segments in the report tree.
- An entry that matches no existing, non-ignored directory gives a `scan/scope-unmatched` warning naming the entry. The hint depends on why (N12): an ignore file or config pattern → the git idiom in `ignore` (`!dist/`, `dist/*`, `!dist/vendor/`); a hard ignore (marker file, `outDir`, skipped package) → names it, since `!` can't revive those; no match at all → `scope` entries are root-anchored globs, so `src` matches only the root `src` (try `**/src`; N16).
- **What `scope` does not limit** (D101: metadata is read wherever it lives): workspace detection, manifests, READMEs, licences, config files, test configs, and every `RepoFiles` method (§7).
- **Units outside `scope` (Q4, D106):** dropped **by the CLI** after `analyze` (C9 helper, reused by C11), not by every module. Modules need no knowledge of `scope`, and third-party modules get it for free. Order (M5): first the D70 check that each unit directory exists on disk (a nonexistent one is still a module error), then units whose directory isn't scanned (`RepoScan.isScanned(unit.dir)` false: out of scope or ignored) are dropped with an info log line each, then the remaining D70 checks. The filtered analysis is what every later call receives (M6, C9). `analyze` issues whose `dir` isn't scanned are dropped the same way, with an info log line (N17, D101's "without an issue"); issues without a `dir` (package or repo level) are kept.
- **Packages:** a package stays in `RepoScan.packages` when at least one of its directories is scanned. A package whose every directory is ignored or outside `scope` is dropped with an info log line (like `skip`, D69).

## 6. The walk

- One walk **from the root**, not per package (01a): packages are only owners of directories (`DirectoryInput.package` = the deepest package directory containing it). Nested `.gitignore` files inside packages apply naturally.
- Directory entries are read once (`readdir` with file types) and cached for `RepoFiles.list`.
- Each scanned directory:

```ts
export interface ScannedDirectory {
  readonly path: RepoPath; // '.' for the root
  readonly package?: RepoPath;
  readonly files: readonly string[]; // non-ignored file names
  readonly directories: readonly string[]; // non-ignored subdirectory names, including symlinked ones (never walked, so not scanned)
}
```

- **Order:** depth-first, entries sorted by UTF-16 code unit (`<`), never locale-dependent, so reports are identical on every machine.
- **Symbolic links (M10):** a symlink to a directory is listed (in `directories` and by `list`, with `symlink: true`) but never walked (no cycles, nothing outside the root), so it is never scanned; a symlink to a file is a file. Broken links, FIFOs, sockets and devices are skipped everywhere.
- Unreadable directories (permissions) get a `scan/unreadable` warning and are skipped; entries vanishing mid-walk are skipped silently (M20, §10). Both are tested with a mocked `fs`.

## 7. `RepoFiles`, `DirectoryInput` and read tracking (D40, D108, K10)

**One rule for every access method (D108):** by default an ignored path doesn't exist; `{ ignored: true }` is the explicit way past that. The same signatures on `RepoFiles` and `DirectoryInput` (a scoped view of it), changing C2 §5.3 and §5.5 (amendment B4, below):

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
- **`DirectoryInput.files` / `directories`** (the eager listing `verify` gets) stay **filtered names**: they describe the directory *as scanned*, which is what the requirement hands to verification ("sans any ignored via `.gitignore`"). The full picture is `dir.list(undefined, { ignored: true })`.
- Paths are repo-relative POSIX paths (or, on `DirectoryInput`, relative to the directory, with `resolve()` to repo-relative). The boundary is **lexical** (M10, B4): a path that normalises outside the root is rejected; symlinks inside the root are followed (`read` of a symlinked file reads its target, as `fs` does; N7), so the boundary is a guard against mistakes, not a sandbox (C2 principle 7).
- Tools that read the disk directly (the TS compiler in `analyze`, Vitest, Vite) still see `node_modules` and build output, as after `npm install`.
- **No `isIgnored(path)`** (D108): `exists` with and without the option answers it; it can be added later, additively (D13).
- **Read tracking with `AsyncLocalStorage`:** the CLI runs each consumer inside a tracking scope; every `read`, `exists` and `list` made inside it, through **any** `RepoFiles` or `DirectoryInput` object, with or without `ignored: true`, is recorded for that scope. That attributes `verify`'s reads through `ctx.repo.files` to its directory too, which per-object tracking would miss. Reads of ignored files are tracked like any other, so C11 watches them.

```ts
export interface TrackedRepoFiles extends RepoFiles {
  /** Runs `fn` and returns its result together with everything it accessed. */
  track<T>(fn: () => Promise<T>): Promise<{ result: T; accessed: readonly FileAccess[] }>;
  /** dev (C11): drops cached contents and listings for these paths so the next access re-reads them. */
  invalidate(paths: readonly RepoPath[]): void; // also drops the cached listing of each path's parent, so adds and removes show (N8)
  directory(dir: ScannedDirectory): DirectoryInput; // its eager files/directories count as a `list` of the directory for tracking (N8)
}

export interface FileAccess {
  readonly kind: 'read' | 'exists' | 'list'; // a 'read' changes on edit; 'exists' and 'list' change on add/remove
  readonly path: RepoPath;
}
```

C4 provides the mechanism; matching watcher events to recorded accesses and re-running `verify` is C11's (K10). **`readJson`:** not added (C2 §5.3).

## 8. Manifests, READMEs and licences

- **`PackageInfo`** from `package.json`: `source: 'npm'`, `manifest`, `name`, `version`, `description`, `license` (as declared, also the legacy `{ type }` object form), `homepage`, `keywords` (strings only), `private`, `raw` (the parsed manifest).
- **Invalid package manifest** (unparseable JSON, not an object): the package is kept with `PackageInfo` reduced to `source` and `manifest`, plus a `scan/invalid-manifest` error on that package. An invalid **root** manifest is fatal in either layout (M9).
- **README:** the non-ignored file in the package (or root) directory matching `readme.md` case-insensitively; an exact `README.md` wins over other casings. Only markdown READMEs; `README` and `README.txt` are not looked up (C5 renders markdown READMEs).
- **Licence (D12):** non-ignored, case-insensitive `LICEN[CS]E` with an optional `.md` / `.txt` extension. Several matches: no extension first, then `.md`, then `.txt`; with the same extension, `LICENSE` before `LICENCE`, then upper case before other casings (M15). The `Doc.format` (D71) follows the extension; C5 renders it.
- **Single package:** the package's `readme`/`licence` are the root ones (C2 §4.3). In workspaces, packages have only their own (no inheritance, C2 §4.3).
- READMEs and licences of packages are found even when the package directory is outside `scope` (metadata, D101).

## 9. VCS information (D24, D110)

**No git at all (N9, D110).** VCS information comes only from values the user wrote down: the root `package.json#repository`, overridden by config `vcs.*` keys and their flags. The CLI never runs `git` and never reads `.git` (no binary dependency, no remote or branch guessing, no inherited-environment or ownership pitfalls).

| Setting | Flag | Meaning | Default |
| ------- | ---- | ------- | ------- |
| `vcs.webUrl` | `--vcs-web-url` | the repository's web page (host, owner, repo) | parsed from the root `package.json#repository` |
| `vcs.directory` | `--vcs-directory` | where the scanned root sits inside that repository (monorepo subdirectories) | `package.json#repository.directory`, else none |
| `vcs.host` | `--vcs-host` | optional: `github`, `gitlab` or `bitbucket`, for self-hosted instances (GitHub Enterprise, self-hosted GitLab, Bitbucket Server) whose URL layout is the same | detected from the host name of `webUrl` |
| `vcs.fileUrlTemplate` | `--vcs-file-url-template` | the full file-link pattern, for hosts we don't know | derived from the host of `webUrl` |

Precedence per setting: flag → config → `package.json`. The keys and flags themselves are C6's; C4 takes the resolved overrides as `ScanOptions.vcs` and produces `VcsInfo`.

- **Parsing `package.json#repository`** uses npm's own `hosted-git-info` (the same data npm uses for the "Repository" link on npmjs.com): object (`url`, `directory`) and string shorthand forms (`owner/repo`, `github:`, `gitlab:`, `bitbucket:`), URLs in `https://`, `git+https://`, `git+ssh://`, `git://`, `ssh://` and scp-like forms (M7). **Credentials in URLs are stripped** and never logged or stored. Unknown hosts: `webUrl` is the URL with `git+`, credentials and `.git` removed, host `other`.
- **Host:** `vcs.host` when set; otherwise `github.com` → github, `gitlab.com` → gitlab, `bitbucket.org` → bitbucket, anything else → `other` (no file links unless `vcs.host` or `vcs.fileUrlTemplate` is set).
- **The branch is always the literal `HEAD`** (D110, revising D105), which GitHub, GitLab and Bitbucket all resolve to the default branch. A template can pin a branch.
- **`VcsInfo.fileUrlTemplate`** is fully resolved except `{path}` (relative to the scanned root), which C5 fills (URL-encoded per segment):
  - github: `<webUrl>/blob/HEAD/<directory/>{path}`
  - gitlab: `<webUrl>/-/blob/HEAD/<directory/>{path}`
  - bitbucket: `<webUrl>/src/HEAD/<directory/>{path}`
  - other: none, unless `vcs.fileUrlTemplate` is set.

  Directories work with the same URLs (GitHub and GitLab redirect `blob` to `tree`; Bitbucket's `src` serves both).
- **A user-supplied `vcs.fileUrlTemplate`** is used verbatim with `{path}` as its only placeholder; `vcs.directory` doesn't apply to it (it contains any prefix and branch itself).
- **Nothing known** (no `repository`, no config): no `VcsInfo`; repo-file links aren't turned into host URLs (C5 decides what they become) and `build` warns once (C11), naming `repository` and `vcs.webUrl`.
- `VcsInfo.kind` stays `'git'` (every supported host is a git host); `VcsInfo.defaultBranch` goes (B4).

Examples (a README links to `src/Button/index.ts`):

| Inputs | Link |
| ------ | ---- |
| `repository: { url: 'git+https://github.com/ofzza/stdblox.git' }` | `https://github.com/ofzza/stdblox/blob/HEAD/src/Button/index.ts` |
| `repository: { url: 'https://github.com/acme/mono.git', directory: 'apps/lib' }` | `https://github.com/acme/mono/blob/HEAD/apps/lib/src/Button/index.ts` |
| the same without `directory`, plus config `vcs: { directory: 'apps/lib' }` | the same |
| config `vcs: { webUrl: 'https://gitlab.com/acme/widgets' }` | `https://gitlab.com/acme/widgets/-/blob/HEAD/src/Button/index.ts` |
| config `vcs: { webUrl: 'https://gitlab.acme.internal/acme/widgets', host: 'gitlab' }` | `https://gitlab.acme.internal/acme/widgets/-/blob/HEAD/src/Button/index.ts` |
| config `vcs: { webUrl: 'https://git.acme.internal/acme/widgets', fileUrlTemplate: 'https://git.acme.internal/acme/widgets/src/branch/main/{path}' }` | `https://git.acme.internal/acme/widgets/src/branch/main/src/Button/index.ts` |

## 10. Errors and scan issues

- **Thrown (fatal, the action stops):** root missing or not a directory; Yarn PnP; an unparseable root manifest, in either layout (M9: it can't be told whether it declared a workspace); an explicit `outDir` that is the root, an ancestor of it, or contains a package (M13; C6 validates this early too); Messages name the path and say what to do.
- **Scan issues** (returned in `RepoScan.issues`, shown by `check` like any other issue, overridable through `rules`): `scan/no-manifest` (warning), `scan/invalid-manifest` (error, on the package), `scan/scope-unmatched` (warning), `scan/unreadable` (warning, on the directory, which is then skipped; M20).
- **Vanishing entries:** a file or directory deleted between listing and reading (`ENOENT` mid-walk, typical in `dev`) is skipped silently (M20). The rule ids are documented in the CLI README (C9 lists them with the module's rules).
- **Log lines only:** dropped packages (§5), dropped units and issues (§5), running inside a workspace package.

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
  readonly linkNodeModules?: boolean; // symlink the fixture's installed node_modules into the copy (default false)
  readonly files?: { readonly [path: string]: string }; // extra files written relative to the copy (content a fixture can't commit, e.g. under its own .gitignore); `../x` lands in the temporary parent directory, so a test can put a `.gitignore` above the scan root (N10)
}

export interface FixtureCopy {
  readonly dir: string; // absolute; the fixture root, `<tmp>/fixture/`
  cleanup(): Promise<void>; // removes `<tmp>`
}
```

- Finds `fixtures/` from its own location (`import.meta.url`), so it works from any package's tests.
- Copies into `<tmp>/fixture/`, with `<tmp>` from `fs.mkdtemp(os.tmpdir()/tsshowcase-fixture-)`, skipping `node_modules`. No git setup: the product never uses git (D110); it can be added when a later chunk needs it.
- Own tests: copying, `node_modules` skipped, `files` (including `../`), cleanup.
- The N3 differential test calls `git check-ignore` (test tooling only) with every inherited `GIT_*` variable stripped and `GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_NOSYSTEM=1` (N11), so the developer's git config can't leak in; it needs a `git` binary (CI has one).

### 11.2 Fixtures added by C4

| Fixture | Purpose |
| ------- | ------- |
| `single-basic` | One package: `repository` field, `README.md`, `LICENSE`, `src/` with a few nested directories, a nested `.tsshowcaseignore` with a `!` re-include, an ignored directory with a file worth re-including through config `ignore`, an empty directory (`.gitkeep`-free, created by the test), a symlinked directory (created by the test) |
| `workspace-npm` | `workspaces: ['packages/*', '!packages/ignored']`, a root README and licence, package `a` (`README.md`, `LICENSE.md`), package `b` (lower-case `readme.md`, no licence), a matched directory without `package.json`, a package ignored by `.tsshowcaseignore` |
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
- scope: default, plain paths, globs (incl. dot directories), ancestors not scanned, units dropped in the M5 order, `outDir` validation, unmatched warning (and the `ignore` hint), packages dropped, `isScanned`, validation errors;
- workspace detection: each manager, negations, directories without manifests, pnpm precedence and a settings-only `pnpm-workspace.yaml`, PnP error, unparseable root manifests, inside-a-workspace-package info;
- `RepoFiles`/`DirectoryInput`: `read`/`exists`/`list` with and without `ignored: true` (entries, `symlink`, reasons and their priority), eager listings filtered, caching, root escape rejected, relative and parent paths, the eager listing tracked as a `list` and parent listings invalidated (N8), `list` of an ignored directory rejecting (N14), `track()` attribution under concurrent scopes (two interleaved `verify`-like calls, including reads with `ignored: true`), `invalidate`;
- manifests, README casing, licence precedence, single-package collapse;
- VCS: every `repository` form and host (incl. `git+https`, `git+ssh`, `git://`, shorthand), credential stripping, `directory` from `package.json` and from config, config/flag precedence, `vcs.host` on a self-hosted URL, a user template used verbatim, nothing known → no `VcsInfo`, every row of the §9 examples; no `git` process is ever spawned (asserted);
- determinism: the same fixture scanned twice gives deeply equal results, and the order doesn't depend on locale.

## 13. Documentation

- CLI README: how the repo is scanned (root, what exists: hard ignores, `.gitignore` and `.tsshowcaseignore`, config `ignore` and its priority (with the git idiom for re-including inside an ignored directory), the gitignore-syntax surprise; `scope`; workspaces; the marker file; VCS settings (`package.json#repository`, `vcs.*`, always `HEAD`, no git); Yarn PnP), and the `scan/*` rule ids. The config keys themselves are documented with C6.
- `-api` docs for `RepoFiles`/`DirectoryInput` (B4): the "ignored doesn't exist unless asked" rule and why consumers that discover files should not pass `ignored: true` casually.
- `-test` README; CONTRIBUTE.md: `copyFixture`, the `broken-*` fixture naming, fixtures using `.tsshowcaseignore` and `!` re-includes against this repo's `.gitignore`. The C1 §5 gotcha text points at the helper.
- AGENTS.md trigger table: a new fixture row ("new fixture → fixtures README table").

## Acceptance criteria

- `detectWorkspace` and `scanRepo` produce the documented results for every fixture, deterministically.
- `toRepoInput` output satisfies the C2 `RepoInput` type (as amended by B4), and `directory()` the `DirectoryInput` type (type tests).
- Read tracking attributes accesses to the right scope under concurrency.
- No credential from a `repository` URL appears in `VcsInfo`, logs or errors.
- `npm run ci` passes, including Prettier with the `broken-*` fixtures and the `-test` package's own tests.

## Out of scope

Config loading, per-package config files, `scope`/`ignore` key and flag parsing (C6); dropping units outside `scope`, the report (C9); matching watcher events to tracked accesses, the marker file's contents and writing (C11); filling `{path}` and rendering links (C5); analysis (C8).

## Recorded for later chunks

- **C5:** `{path}` is the only placeholder, URL-encoded per segment; no `fileUrlTemplate` (no VCS info, or host `other` without a template) → decide what a repo-file link becomes. Links to dropped units (out of scope) and to ignored files need an explicit mapping (a VCS URL 404s for a gitignored file) (M6).
- **C6:** calls `detectWorkspace` first, then loads per-package configs from `WorkspaceInfo.packages`, then `scanRepo`; config keys `scope` and `ignore` with repeatable `--scope` / `--ignore` flags (D107); flags vs config arrays and their base directory (M12, recommendation: `--ignore` appends, `--scope` replaces, both relative to the repo root); an unknown `include` or `exclude` key gets a "did you mean `scope`/`ignore`?" error; validates `scope` with `validateScope()`; validates an explicit `outDir` early (M13); error for a package-level config at the root (L4); `scan/*` and `module/error` are CLI-known rule ids for `rules` validation (M15); attaches merged options with `toRepoInput`.
- **C7, C10:** discovery through `RepoFiles.list` without `ignored: true`; `DirectoryListing` is `entries` (B4); tests in out-of-scope directories still run and end up unmapped (M6).
- **C8:** module-side memoisation (a promise cached in `state` during `analyze` and awaited in `verify`) defeats `AsyncLocalStorage` read tracking; document it for module authors (C13) and avoid it in `-base-react` (M11). Fixtures with a committed `dist/` re-include it against this repo's `.gitignore` with `!` rules (C1 amendment A3, D109).
- **C9:** report tree with non-scanned ancestors as plain path segments; `scan/*` issues; the M5 order for dropping out-of-scope units (D106), producing the filtered analysis every later call receives; `analyze` issues whose `dir` isn't scanned are dropped with an info log line, issues without a `dir` kept (N17); this repo's config `ignore: ['/fixtures/']` (anchored, N13) (the `-test` package skips itself).
- **C11:** the marker file `.tsshowcase-output.json` (contents, writing; no build-machine paths in it, since it is deployed); `track()` results drive dev re-verify, including reads of ignored files; `invalidate()` on watcher events; a change to any `.gitignore` at or below the root, the marker file or the config changes the verdict for whole subtrees → a full rescan and a new `RepoScan` (config → restart) (M11); groups outside scope in the hierarchy and D78 group READMEs (M6).
- **C14:** `-test` is private; the publish script skips it.
- **C5 (tests):** confirm `HEAD` links for every host, Bitbucket's `src/HEAD/…` included.
- **C6 (D110):** `vcs.webUrl`, `vcs.directory`, optional `vcs.host`, `vcs.fileUrlTemplate` keys with `--vcs-web-url`, `--vcs-directory`, `--vcs-host`, `--vcs-file-url-template`; `vcs.defaultBranch` removed from the schema.
- **C11:** the one-time `build` warning when READMEs link to repo files and no VCS info is known.
- **C6/C13 docs (N16):** `scope` uses root-anchored globs, `ignore` gitignore syntax (unanchored without a slash); the asymmetry is documented next to both keys.

## C2 amendment B4 (re-approved with this plan, applied)

- **§5.3 `RepoFiles`:** `read`/`exists`/`list` take `options?: { ignored?: boolean }`; by default an ignored path doesn't exist (read rejects as missing, `exists` is false, `list` leaves it out), `ignored: true` sees ignored paths too, except `.git` and `node_modules`, which are never reachable (D108, M21). `DirectoryListing` becomes `{ entries: DirectoryEntry[] }` with `DirectoryEntry { name, type, symlink, ignored, reason? }`. The comment "non-ignored entries only" becomes the default-behaviour note, and the JSDoc's root-escape sentence becomes "lexical boundary, symlinks followed" (N6).
- **§5.5 `DirectoryInput`:** the same three signatures; `files`/`directories` stay filtered names (the directory as scanned). The scan sentence: `.gitignore` and `.tsshowcaseignore` files at or below the root (not above it, not `.git/info/exclude`), config `ignore` patterns (overriding `.gitignore`, `!` included), directories outside `scope`; `exclude`/`include` renamed (D107).
- **§4 `VcsInfo`:** `defaultBranch` removed; `fileUrlTemplate` documented as fully resolved except `{path}` (D110).
- **§10 config sketch:** `vcs` becomes `{ webUrl?, directory?, host?, fileUrlTemplate? }`, `defaultBranch` removed (D110); `include` → `scope` ("where to look: the directories checked and showcased; never un-ignores"), `exclude` → `ignore` ("gitignore syntax incl. `!`; overrides `.gitignore`").
- **Also stale in C2 (M3):** the §5.3 comment block under `RepoFiles` ("the gitignore-aware view … plain `fs` would descend into ignored directories") rewritten for the default/opt-in rule, and `list(dir)` takes an optional path; §5.4's D72 paragraph and §5.5's opening ("every non-ignored directory") become "every scanned directory (in `scope`, not ignored)"; §5.5's "the repo root (`'.'`) is verified too" becomes "when it is in scope"; the root-escape sentences become lexical with symlinks followed (M10); the §4 `VcsInfo.fileUrlTemplate` example becomes `'https://github.com/o/r/blob/HEAD/{path}'` with a note that `{path}` is the only placeholder (D110).

## Super-plan R5 and AGENTS edits (applied)

- **R5 (C4 planning, D103–D110):** C4's "Git info: remote → host, default branch … fallback" bullet → VCS from `package.json#repository` and config only, no git (D110); C4 bullets: `.gitignore` files at or below the root only, plus the `.tsshowcaseignore` alias (no `.git/info/exclude`), config `ignore` overrides with `!`, hard ignores, `scope`, scanner inputs `{ scope, ignore, skippedPackages, outDir }`, the `-test` package. C6: `scope`/`ignore` keys and flags. C8: "units outside `include` are dropped" moves to the CLI (C9, D106). C9: the drop in the duties table, and "tree report of every non-ignored directory" → "every scanned directory" (N15). Package table and C1's package list: `-test` (private). D27's kinds gain `test`.
- **C1 amendment A3 (M4):** §1 layout and §2 package and dependency tables gain the private `-test` package (devDependency only, no project reference); §6 changesets `privatePackages: { version: false, tag: false }`; §5 "C4 also decides where the helper lives" → "lives in `-test` (C4)"; "Recorded for later chunks" C9: `exclude` → `ignore`; §5 "Prettier skips only their `node_modules`" → also `fixtures/broken-*/`; §5 "must be excluded via this repo's tsshowcase config" → "ignored via config `ignore`"; §5 gotcha 2 notes that the root `.gitignore` no longer leaks into fixture scans (M19), while git state still does (N15); §5 states that fixtures needing content this repo's `.gitignore` would catch re-include it with `!` rules (at the root, in `fixtures/.gitignore` or per fixture), and that fixtures use `.tsshowcaseignore` for their own ignore rules (M2, D109).
- **AGENTS:** D103–D110 added (D110 revises D24 and D105); D101's text updated for the renames and for `ignore` deciding what exists, its status "revised by D103, D106, D107"; §1 summary ("all directories not ignored by `.gitignore`") and §1's "receives every non-ignored subdirectory" note the config `ignore` and `scope` (N15); D27, D14 mention `-test`; D41 notes `-test`'s `skip`; open points: C4 removed, C6/C8/C9 rows updated (`scope`, `ignore`, D106, `fixtures/` via `ignore`), plus the M6/M11/M12 points; C4 marked approved in the tracker; §6 pitfalls unchanged.

## Open questions

- ✅ **Q1 What exists (D103, D107, D108, D109):** `.gitignore` files decide, regardless of git and of tracking (no `.git/info/exclude`, no global excludes, only `.gitignore` files at or below the root, M19); config `ignore` (renamed from `exclude`) overrides them on the same path, `!` included, under git's parent rule like every source (N1); hard ignores stay. A `.tsshowcaseignore` alias acts as lines appended to the same directory's `.gitignore` (D109, from M2). `scope` (renamed from `include`) only says where to look. Every access method treats ignored paths as nonexistent unless called with `{ ignored: true }`; `list` returns `DirectoryEntry` descriptors. (Discussed over several rounds: Claude first recommended JS rules with `.git/info/exclude`, then a git-backed "fresh clone" model; the user settled on `.gitignore` only, a separate `ignore` override and the rename, and asked for one consistent access rule.)
- ✅ **Q2 `copyFixture` home (D104):** a private `-test` package (the user's choice after weighing a root `test-utils/` directory; named like `-lint`).
- ✅ **Q3 Branch fallback (D105, revised by D110):** `origin/HEAD`, else the literal `HEAD` in URLs; since N9, always `HEAD` (no git).
- ✅ **Q4 Units outside `scope` (D106):** dropped by the CLI after `analyze`, in the M5 order (existence check, drop, remaining D70 checks); the C8 open point changes accordingly.
