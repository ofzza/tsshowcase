# Chunk plan C8: React base, Node side (`@ofzza/tsshowcase-base-react`, `@ofzza/tsshowcase-module-react-default`)

> **HISTORICAL: superseded by `08b-representation-redesign.md`; this plan will be rewritten on the representation model.** Its proposed amendments B9 (C2) and R10 (super-plan) are void; the redesign uses those ids for its own amendments (08e E7).

> Status: **DRAFT** (round 2, 2026-10-19). Round 1's questions Q1–Q8 are resolved (D149–D159); they reshaped the unit rules (declaring directory + claims), added CLI-owned directory configs (`.tsshowcase.*`), per-node `showcase`/`name`/`slug`/`order`, broadened groups and declaration-keyed kind overrides. These need amendments to C2 (B9), C4, C6, C9, C10, C11 and super-plan R10, listed at the end. Decisions referenced as `D<n>` are in AGENTS.md §3.

## Goal

Everything `check` and `init` need for React/TS repos, with no bundler involvement (D48):

- the **static entry analysis** (D5): package entries → public symbols → units (D153), kinds (D158), showcases (D149), group showcases (D157);
- the **base rules** (D21, D37, D38) and the opt-in **`headless` preset** (D47, D159), as `verify`;
- the **scaffold** for `init` (D16, D150);
- the **preliminary React showcase file format** (D96, D149), finalised in C10;
- the first version of the **base module class** (`analyze`, `verify`, `scaffold`), plus the loadable **`react-default` class** extending it, so C9 can run `--module react-default` before C10 exists.

C10 later adds `resolveRoute`, `runTests`, `build` and `dev` to the same classes. The **directory configs** themselves (D154) are a CLI concept: the file format, loading and validation are C6's, collecting them is C4's (amendments below); C8 applies their unit semantics.

## Requirements this plan answers

| Source | Point | Where |
| ------ | ----- | ----- |
| Super-plan C8 | entry analysis, dist → source, the repo's own TypeScript, external re-exports, kinds | §3, §4 |
| Super-plan C8 | base rules, `headless`, rule ids and severities | §6 |
| Super-plan C8 | scaffold templates; showcase files kept out of the user's library build | §7 |
| D96, 01d L6 | preliminary showcase file format | §5 |
| 01a | `init` semantics | §7 |
| 01b K19 | TypeScript 7 has no stable programmatic API | §3.1 |
| C4 M11 | module-side memoisation defeats read tracking | §6.4 |
| C1 A3 | fixtures with a committed `dist/` | §9 |
| D116, D136, D138, D145, D148 | the options schema | §8 |

## 1. Where the code lives

```
packages/tsshowcase-base-react/src/
  index.ts                     re-exports the units
  ReactModuleBase/             unit: the base module class (analyze, verify, scaffold; C10 adds the rest)
  ReactModuleOptions/          unit: options.schema.json + sidecar, the inferred ReactModuleOptions type, OPTION_FLAGS
  ShowcaseProps/               unit (type-only): the props a React showcase receives (§5)
  rules/                       unit: REACT_RULES, the RuleDescriptor list (ids, descriptions, default severities, presets)
  _analysis/                   internal: TypeScript loading, entry resolution, the export walk, units and claims, kinds
  _verify/                     internal: base rules, the headless preset, showcase file checks
  _scaffold/                   internal: init planning and templates
  _showcase-format/            internal: file naming, name derivation, the syntactic export checks

packages/tsshowcase-module-react-default/src/
  index.ts                     re-exports the unit
  ReactDefaultModule/          unit: `class ReactDefaultModule extends ReactModuleBase` (C8: display name only; C10: layout, build, dev)
```

- Internal directories are `_`-prefixed (D10), have tests, and are not units (D41 applies to exported units). They are neither units nor groups (§6.3), so the dogfood check passes.
- `react-default`'s package default-exports the class (D9) and also exports it by name.
- **Dependencies of `-base-react`:** `@ofzza/tsshowcase-api`, `@typescript/typescript6` (§3.1), `semver`, `picomatch` (claim patterns, §4.3). No React and no Vite in C8 (C10 adds them).

## 2. The class

```ts
export class ReactModuleBase implements ShowcaseModule {
  static readonly apiVersion = '^0.1';
  static readonly displayName: string = 'react-base'; // layout modules override it
  static readonly optionsSchema = optionsSchema; // §8
  static readonly rules = REACT_RULES; // §6
  static readonly optionFlags = OPTION_FLAGS; // §8

  protected readonly options: ReactModuleOptions; // the root options (ModuleInit.options)
  protected readonly logger: Logger;
  constructor(init: ModuleInit<ReactModuleOptions>);

  analyze(ctx: AnalyzeContext): Promise<AnalysisResult>; // §4
  verify(dir: DirectoryInput, ctx: VerifyContext): Promise<VerifyResult>; // §6
  scaffold(ctx: ScaffoldContext): Promise<ScaffoldResult>; // §7
  dispose(): void; // drops the per-instance TypeScript cache (§3.1)
}
ReactModuleBase satisfies ShowcaseModuleConstructor<ReactModuleOptions>;
```

- **Not loadable on its own** (super-plan): `-base-react` has no default export. `ReactModuleBase` is concrete, so layout modules extend it without boilerplate; C10 decides whether its layout hooks become abstract.
- Static members are inherited: `class ReactDefaultModule extends ReactModuleBase { static override readonly displayName = 'react-default'; }` is a complete C8 module.
- **No per-call state on `this`** (C2 §5.1). The only instance state is a cache of loaded TypeScript instances (§3.1), which holds no repo data.

## 3. TypeScript

### 3.1 Which TypeScript (01b K19, D152)

Per package, `analyze` resolves `typescript` with `createRequire` from the package directory, then from the repo root (hoisted installs):

| Found | Used | Log |
| ----- | ---- | --- |
| `>=5.0 <7` | the repo's own | debug line with version and path |
| `>=7` | the bundled TS 6 API (`@typescript/typescript6`) | one warning per run: "TypeScript 7 has no stable programmatic API yet; analysing with TypeScript 6.0" |
| `<5.0` | the bundled TS 6 API | one warning per run: "TypeScript <x> is older than 5.0; analysing with TypeScript 6.0" |
| none | the bundled TS 6 API | one info line |

- **Facts (checked 2026-10-19):** `typescript@latest` is 7.0.2, whose package exports only `lib/version.cjs` and `unstable/*` APIs; a stable API is expected in 7.1. `@typescript/typescript6@6.0.2` (published by the TypeScript team) has `main` → the 6.x JS API. A plain `typescript@6` dependency would be hoisted next to, or clash with, the repo's own `typescript` in the CLI's install tree; the scoped name never does.
- TS ≥ 7 is detected from `typescript/package.json#version` without importing it.
- Loaded instances are cached per resolved path on the module instance; `dispose()` drops the cache.
- Recorded for later: use TS 7's own API once it is stable (7.1).

### 3.2 Which tsconfig

- The package's `tsconfig.json`, else the nearest one above it up to the repo root, else defaults (`allowJs`, `jsx: preserve`, `moduleResolution: bundler`). The `tsconfig` option (§8, per package) overrides the lookup.
- Read with `ts.getParsedCommandLineOfConfigFile` (comments, `extends`). Its `fileNames` are **not** used: the program's root files are the resolved entry sources only (§4.1).
- Overrides on top: `noEmit`, `allowJs`, `skipLibCheck`, and the `conditions` option (§8) prepended to `customConditions` when `moduleResolution` supports it (`node16`, `nodenext`, `bundler`).
- Diagnostics are not reported: analysis is not type-checking.

## 4. `analyze` (D5)

For every package in `ctx.repo.packages` (checking `ctx.signal` between packages):

### 4.1 Entries, and dist → source

1. **Entry list** from `package.json`:
   - `exports`: a string or a `'.'`-less condition object means `'.'`; otherwise every subpath key. Subpath **patterns** (`'./*'`) are not expanded: a `react/entry-pattern` warning suggests listing them in the `entries` option. Subpaths mapped to `null` are skipped.
   - Without `exports`: `types`/`typings`, `module`, `main`, in that order, as `'.'`.
   - Without any of these: `src/index.{ts,tsx,mts,js,jsx,mjs}`, then `index.{…}` at the package root.
   - The `entries` option (§8, per package, `{ subpath: path }`) replaces the list entirely.
2. **Target per subpath:** the first match in condition order: the `conditions` option (default `['source']`), then `types`, `import`, `module`, `default`, `node`, `require`; nested condition objects recursively; arrays take their first string.
3. **Map to source** when the target isn't a TypeScript source file (`.ts`/`.tsx`/`.mts`/`.cts`, not `.d.ts`):
   - through the tsconfig's `outDir` (and `declarationDir` for `.d.ts`) → `rootDir` (when unset: `src/` if it exists, else the package dir), swapping the extension (`.js` → `.ts`, `.tsx`; `.mjs` → `.mts`; `.d.ts` → `.ts`, `.tsx`);
   - else the target itself, if it is a JS source file (JS packages, `allowJs`).
   - A candidate counts only if `ctx.repo.files.exists()` says so: **ignored = nonexistent** (D103), so a gitignored `dist/` is never analysed and units never land where D70's existence check would reject them.
4. A subpath with no usable source gets a **`react/entry-unresolved`** error on the package (hint: add a `source` export condition, or set `entries`). A package with no resolvable entry at all yields `units: []` (still visible, D72) plus that error.

**`unitRoot`** = the nearest common directory of the package's entry source files (e.g. `src`), else the package directory. It never widens to fit stray symbols (§4.3).

### 4.2 The export walk

One `ts.Program` per distinct (TypeScript instance, tsconfig) pair, with every entry source of the packages sharing it as root files. For each entry:

- `checker.getExportsOfModule(entrySymbol)` gives the public names; each is resolved to its declaration with `getAliasedSymbol` (all the way). A **public declaration** is its (declaring file, declared name) pair; `default` for an anonymous default export.
- **External re-exports are never units:** a declaration in a file outside the package directory (an npm dependency, a sibling workspace package reached through `node_modules`, a library `.d.ts`) is skipped, logged at debug level. (Units outside `scope` are the CLI's to drop, D106.)
- `ExportedSymbol.entry` is the subpath the symbol is reachable from; a symbol reachable from several entries appears once per entry.

### 4.3 Units and claims (D153)

**The base rule:** a public declaration's unit is the **directory of its declaring file**. Barrels don't matter, except that the entry must reach the declaration. Nesting follows directly (D36): `src/Card/CardHeader/CardHeader.tsx` is the unit `src/Card/CardHeader`, inside the unit `src/Card`.

**Claims (directory configs, D154):** the CLI hands the module every directory config, from `.tsshowcase.*` files and from the `units` maps of the root and package configs, normalised to repo-relative paths (`RepoInput.overrides`, C2 B9). A directory config's `scope` (patterns relative to its directory, downwards only) claims declaring files below it; a directory config also implicitly claims the files directly in its own directory.

**Resolution, for a declaration in file `F` in directory `D`:** walk from `D` up to `unitRoot` (inclusive); the **first directory config that claims `F`** decides the unit. With no claim anywhere, `D` is the unit. So an ancestor's claim beats a plain `D`, never a `D` with its own directory config; some children can be units while others hand off to a parent or grandparent, without conflicts.

- **The unit root can be a unit** (a flat `src/*.tsx` library is one unit at `src`), implicitly when the entry's directory declares symbols, or explicitly. A stray `export const VERSION` in `src/index.ts` thus makes `src` a unit that must meet the unit rules, so it is still reported.
- **Declarations outside `unitRoot`** (inside the package, e.g. `lib/format.ts` re-exported from `src/index.ts`) → **`react/outside-unit-root`** error on the file (hint: move it under `<unitRoot>`); no unit. Claims can't reach it (they only reach downwards).
- **A unit** is a directory with one or more public declarations: `AnalyzedUnit { dir, kind, exports, readme: '<dir>/README.md', showcases }` (§5), sorted by directory.
- **Config consistency** (warnings, module-emitted because only the module resolves symbols): a `scope` pattern that claims no public declaration → `react/unused-claim`; a kind override (§4.4) matching no public declaration → `react/unused-kind`.
- A directory with a directory config but no declarations (own or claimed) is not a unit; it may be a group (D157). If it is neither, the CLI warns that the directory config applies to nothing (C9).

### 4.4 Kinds (D37, D158)

Per declaration, from the checker:

| Kind | Rule |
| ---- | ---- |
| `type` | interfaces, type aliases, and anything only reachable as a type (`export type`), `typeOnly: true` |
| `hook` | a function (declaration or function-valued `const`) named `use[A-Z0-9]…` |
| `component` | PascalCase and one of: a function whose return type mentions `JSX.Element`, `ReactElement`, `ReactNode` or is JSX by syntax; a value whose type has `$$typeof` (`memo`, `forwardRef`, `lazy`); a class extending `Component`/`PureComponent` |
| `class` | any other class |
| `function` | any other function |
| `constant` | everything else (variables, enums, namespaces) |

- The JSX-by-syntax fallback makes classification work without `@types/react` installed.
- **Overrides (D158):** `RepoInput.overrides.kinds`, keyed by declaration (`'<repo-relative file>#<declared name>'`), from `kinds` maps in directory configs (keys relative to their directory) and in the root and package configs (keys relative to the repo root or the package); normalised and checked for duplicates by the CLI. An override wins over the heuristic; its value must be a well-known `UnitKind` (else a `react/invalid-kind` error).
- **Unit kind:** the first kind present in the order `component`, `hook`, `class`, `function`, `constant`, `type` (`type` only when every export is type-only).

### 4.5 Group showcases (D157)

A directory at or below `unitRoot` that is not a unit but holds showcase files (§5) is reported in `PackageAnalysis.groups` (`{ dir, showcases }`, C2 B9), so the CLI can make it a group with showcases (e.g. a "Forms" page composing several units).

### 4.6 Result

- `packages`: `{ dir, unitRoot, units, groups }` per package with a manifest.
- `issues`: the analyze issues above.
- `inputs` (D68): the program's source files inside the repo root and outside `node_modules`, every package's `package.json`, every tsconfig in each `extends` chain inside the repo. (Directory configs are the CLI's own input; C11 re-analyses when one changes.)
- `state`: nothing in C8 (§6.4).
- Showcase discovery uses `ctx.repo.files.list(dir)`, so it respects the ignore rules.

## 5. The preliminary React showcase file format (D96, D149)

- **One showcase per file:** `*.showcase.{tsx,jsx}` directly in a unit or group directory (not in subdirectories, which may be units of their own).
- **`ShowcaseEntry.name`:** an optional `export const name = 'Sizes';` (a string literal, read statically) wins. Otherwise the file name: `<Unit>.showcase.tsx` → the unit name; `<Unit>.<variant>.showcase.tsx` → `<variant>` in title case; any other `<x>.showcase.tsx` → `<x>` in title case. Entries are ordered with `<Unit>.showcase.tsx` first, then by file name.
- **Default export:** a React component receiving the showcase props:

  ```ts
  export interface ShowcaseProps {
    readonly colorScheme: 'light' | 'dark'; // the showcase's selected scheme (D89, D90)
  }
  ```

  How the props are typed is the author's choice: inline, or `import type { ShowcaseProps } from '@ofzza/tsshowcase-base-react'`. Only the shape matters; nothing checks the props type. The scaffold writes it inline, so a repo needs no type dependency on `-base-react`.
- **The `shell` opt-in (D41, D100):** `export const shell = true;` renders the showcase inside the shell (catalogue-styled components only; no scheme selector). Must be the literal `true`.
- Several showcases = several files (D61). C10 finalises the format (and re-approves C8 if it changes).

## 6. `verify` (D2, D21, D37, D38)

### 6.1 Rules

| Rule id | Default | Applies to | Check |
| ------- | ------- | ---------- | ----- |
| `react/entry-unresolved` | error | package (analyze) | §4.1 |
| `react/entry-pattern` | warning | package (analyze) | §4.1 |
| `react/outside-unit-root` | error | file (analyze) | §4.3 |
| `react/unused-claim` | warning | directory config (analyze) | §4.3 |
| `react/unused-kind` | warning | kind override (analyze) | §4.4 |
| `react/invalid-kind` | error | kind override (analyze) | §4.4 |
| `react/showcase-in-build` | warning | package (analyze) | the package's build would compile or publish showcase files (§7) |
| `react/readme-required` | error | unit | `README.md` exists (exact case, D130) |
| `react/test-required` | error | non-type-only unit | a test file: `*.{test,spec}.{ts,tsx,mts,cts,js,jsx,mjs,cjs}` or `*.test-d.ts`, in the unit directory or its `__tests__/` |
| `react/showcase-required` | error | `component`, `hook` units | at least one showcase file (§5) |
| `react/showcase-export` | error | each showcase file (units, groups) | a default export that is a function, a function-valued `const` or a class (syntactic) |
| `react/showcase-shell` | error | each showcase file | a `shell` export, if present, is `export const shell = true` |
| `react/showcase-name` | error | each showcase file | a `name` export, if present, is a non-empty string literal |
| `headless/files` | error | `component` units, preset `headless` | `<X>.types.ts`, `use<X>.ts`, `render<X>.tsx`, `<X>.tsx` |
| `headless/exports` | error | same | the unit exports `<X>`, `use<X>`, `render<X>` |
| `headless/tests` | error | same | `use<X>.test.ts(x)` and `<X>.test.tsx` |
| `headless/showcase-uses-hook` | warning | same | a showcase calls `use<X>(` (a textual match) |

- `<X>` is the unit name (the directory name). Showcase optional for `function`, `class`, `constant`; not applicable to `type`.
- **`showcase: false` (D155) switches off every unit and showcase rule** for that node (README, tests, showcases, the showcase format checks, `headless/*`): the rules exist only to build the showcase, not to enforce library hygiene. Its descendants are unaffected. Analyze issues (entries, claims, kinds) still apply.
- Every issue carries `file` where one exists and a `hint` (usually `run \`tsshowcase init <dir>\``). Severities are defaults; the CLI applies `rules` overrides (D70).

### 6.2 Presets

`presets: ['headless']` (§8, per package). Unknown presets are rejected by the schema's enum.

### 6.3 Non-unit directories (D157)

- **A group** (`VerifyContext.group`, set by the CLI from its hierarchy, C2 B9): passes (`{ issues: [] }`), with the showcase format checks on its showcase files (unless `showcase: false`).
- **Anything else** that isn't a unit (outside `unitRoot`, `_internal`, `__tests__`, asset directories): `{ ignored: true }`, shown greyed ("ignored by module").
- The round-1 `react/orphan-showcase` rule is gone: a showcase in a non-unit directory makes it a group.

### 6.4 Read tracking (C4 M11)

`verify` reads only through `dir` (and `ctx.repo.files`), and takes analysis facts from `ctx.unit`, `ctx.group` and `ctx.repo.overrides`, which are plain data. No promise is cached in `state` during `analyze` and awaited in `verify`. Showcase checks parse with `ts.createSourceFile` on contents read through `dir.read` (syntax only, no program).

## 7. `scaffold` for `init` (D16, D150)

| `<dir>` | `init` does |
| ------- | ----------- |
| a unit | creates its missing stubs: README, a test (non-type units), a showcase (`component`/`hook` units), the `headless` files when the preset is on |
| a directory containing units (a package, a group, `.`) | the same for every unit below it |
| anything else (missing, or not a unit) | **a new unit**: `index.ts`, the source file(s) for the kind, README, test, showcase where required; plus the entry export |

- Units with `showcase: false` get nothing (no rule applies to them, D155).
- **New units:** the name is the directory's base name; the kind comes from `--kind` (`scaffold.kind`, D79), else from the name (`use*` → hook, PascalCase → component, else function). The directory must lie at or below a package's `unitRoot`; otherwise `scaffold` throws a clear error naming the package and its `unitRoot`.
- **The entry export:** `export * from './<relative dir>/index.js';` appended to the `'.'` entry's source file (`ifExists: 'overwrite'`, computed from its current contents). The specifier copies the style of the file's existing relative re-exports (`.js`, `.ts` or extensionless), else `.js` under `node16`/`nodenext` and extensionless otherwise. Skipped with a note when the entry already reaches the new unit's declarations.
- **Never overwrites user files:** stubs are `ifExists: 'skip'`; only the entry edit is `overwrite`. No directory configs are written.
- **Templates:** README (`# <X>` and a placeholder paragraph), tests with `it.todo(…)` stubs importing from `vitest` (no testing library assumed), a showcase rendering the unit with the inline props type, `headless` files when the preset is on. `.tsx`/`.ts`, or `.jsx`/`.js` in packages whose entry source is JavaScript.
- **Keeping showcases and directory configs out of the library (D151):** showcases are never reachable from the entries, so entry-based bundlers skip them. Two leaks remain: a `tsc` build whose tsconfig (`tsconfig.build.json` when present, else the package's tsconfig when it emits) `include`s `*.showcase.*` files without `exclude`-ing them, and a `files` field (or none) that publishes unit directories as source (which also publishes `.tsshowcase.*` files). `analyze` detects both, when showcase files or directory configs exist, and reports `react/showcase-in-build` (warning, on the package, naming the tsconfig or `files` and the pattern to add); `scaffold` adds the same text as a note when it creates a package's first showcase. `init` never edits tsconfig files or `package.json#files`. Bundler configs are not inspected (documented limit).
- **Directory configs and `tsc` (spike, 2026-10-19):** TypeScript 5.0.4 and 6.0.2 wildcard `include`s (`src`, `src/**/*`, `src/**/*.ts`) never match names starting with a dot, so `.tsshowcase.ts`/`.mts` are not compiled unless named explicitly; `npm pack` does publish them when their directory is in `files`. So `react/showcase-in-build` checks `files` for them, and not tsconfig.
- `notes` list what was created, what was skipped, and the build note.

## 8. Options (`ReactModuleOptions/options.schema.json`, D58, D62, D116, D148)

| Option | Type | Scope | Marker | Used by |
| ------ | ---- | ----- | ------ | ------- |
| `presets` | `('headless')[]` | per package | `x-tsshowcase-per-package` | verify, scaffold |
| `entries` | `{ [subpath]: string }` | per package | per-package; values `OPTION_FORMAT.path` | analyze |
| `tsconfig` | string | per package | per-package; `OPTION_FORMAT.path` | analyze |
| `conditions` | string[], default `['source']` | per package | per-package | analyze |
| `scaffold` | `{ kind?: 'component' \| 'hook' \| 'function' \| 'class' \| 'constant' \| 'type' }` | root only | – | scaffold; flag only in practice |
| `testRunner` | string | root only | `OPTION_FORMAT.reference` | C10's `runTests` (D34, D136) |
| `viteConfig` | string | root only | `OPTION_FORMAT.path` | C10 (D138) |
| `showcaseStylesheets` | string[] | root only | items `OPTION_FORMAT.path` | C10 (D90) |
| `showcaseSetup` | string | root only | `OPTION_FORMAT.path` | C10 (D95) |

- Root-only options say "root config only" in their `description` (D136, D145, D148); per-package path values resolve against the package config's directory (D116).
- Kind overrides and unit claims are **not** module options: they are CLI config (`kinds`, `units`) and directory configs (D154, D158).
- C8 declares the C10 options now so the schema, docs and dogfood config don't change shape later; C10 implements them.
- `OPTION_FLAGS = { kind: 'scaffold.kind', 'test-runner': 'testRunner', preset: 'presets' }` (D79).
- Layout modules inherit the schema as is; how a layout module adds options is C10's.

## 9. Fixtures

New fixtures (own lockfile where they install packages, C1 §5):

| Fixture | Purpose |
| ------- | ------- |
| `react-single` | Single package, `typescript@5.x`, `react` + `@types/react`. `exports` with a `source` condition and `dist` targets (no `dist/`). Units: `components/Button` (complete, two showcases, one with `export const name`), `components/Card` with nested `Card/CardHeader` (D36), `hooks/useToggle`, `utilities/format` (function, no showcase), `types/Theme` (type-only, README only), `constants/sizes`; `components/README.md` (a group page); `compositions/Forms.showcase.tsx` (a group with a showcase); `_internal/` with a README and `.tsshowcase` `{ "showcase": false }` |
| `react-claims` | `Buttons/PrimaryButton/parts/Icon.tsx` claimed by `PrimaryButton/.tsshowcase` (`scope: ["parts/**"]`), `Buttons/variants/*` claimed by `Buttons/.tsshowcase.json` while `Buttons/variants/Ghost/.tsshowcase` keeps its own unit; a `units` map in the root config; an unused claim; a `.tsshowcase.ts` with `defineDirectoryConfig`; `name`/`slug`/`order`; a unit with `showcase: false` and no README |
| `react-flat` | `src/*.tsx` only: one unit at the unit root |
| `react-issues` | Every base rule violated once, a declaration outside `unitRoot`, kind overrides (used, unused, invalid) |
| `react-workspace` | npm workspace: `packages/ui` (React; re-exports from `react` and the sibling `@fx/utils`, both external), `packages/utils` (pure TS: functions and types), `packages/dist-only` (exports only to `dist`, mapped back through `rootDir`/`outDir`; a committed `dist/` re-included against this repo's `.gitignore`, C1 A3, ignored by the fixture's `.tsshowcaseignore`), `packages/subpaths` (`'.'`, `'./hooks'`, a `'./*'` pattern), a package config with a `kinds` map |
| `ts-none` | No `typescript`, no tsconfig, plain `.js` sources: the bundled TS 6, `allowJs` |
| `ts-7` | `typescript@7` installed: the fallback warning |
| `react-headless` | One unit following the `headless` split, one breaking each `headless/*` rule, `presets: ['headless']` |
| `react-kinds` | Every export form of §4.4 |

## 10. Tests

- **Tests need a `RepoInput` without the CLI:** the CLI depends on `-base-react` (built-in lookup by name, D9), so `-base-react` depending on it would be a workspace cycle, which `tsc -b` project references refuse; the scanner (C4) is also internal to the CLI. **C4 amendment:** the private `-test` package gains `createTestRepoInput(dir, { packages?, ignored?, overrides? })`: an `fs`-backed `RepoInput` (`node_modules`/`.git` unreachable, `ignored` patterns treated as nonexistent, `PackageInput.options` and `overrides` given by the test). The real scanner and directory-config loading are exercised end to end by C9 (`check --module react-default` over these fixtures).
- Per internal directory: TypeScript selection (each row of §3.1), tsconfig lookup and overrides, entry resolution (every case of §4.1, incl. `dist` mapping and the ignored `dist/`), the unit rule and claim resolution (nearest explicit wins, own config beats ancestor claim, unit root as a unit, outside `unitRoot`), external re-exports, kinds and overrides, group showcases, showcase naming and syntactic checks, every rule of §6 (positive and negative, and switched off by `showcase: false`), scaffold planning for each case of §7, `inputs`.
- `analyze` → `verify` end to end over every fixture against expected units, groups and issues.
- Read tracking: `verify` inside a tracking stub reads every file it depends on through `dir`.
- Type test: `ReactModuleBase` and `ReactDefaultModule` satisfy `ShowcaseModuleConstructor<ReactModuleOptions>`.
- Fixture tests use `copyFixture({ linkNodeModules: true })` wherever resolution matters (§3.1, externals); read-only tests may use `fixturePath()`.

## 11. Documentation

- `-base-react` README: what the package is (not loadable), the export list, the rules table, the showcase file format (preliminary), the options table, the TypeScript selection, the analysis rules (entries, units and claims, kinds and overrides, group showcases) and their limits.
- **The `headless` preset, in detail (D159):** what it is (the `useX` / `renderX` / `X` split and its files), why a library would want it (logic reusable with a custom presentation, presentation testable as a pure function, a thin component, consistency across a component library), how to turn it on, what each `headless/*` rule checks, and **what `check` can't verify**: that `renderX` is pure, that `X` only wires `useX` into `renderX`, that the hook holds all the logic, or that a showcase calling `useX(` really builds a custom presentation (a textual match only).
- `-module-react-default` README: the built-in name; "C8: `check` and `init` only; `build`/`dev` arrive with C10" (removed by C10).
- Unit READMEs per D56. CONTRIBUTE.md glossary: unit root, claim, external re-export (directory config: C6). Fixtures README: the table above.

## Acceptance criteria

1. Over every fixture, `analyze` returns the expected units (dirs, kinds, exports, showcases), groups, `unitRoot` and issues; the result passes D70's checks as amended (units at or below `unitRoot`).
2. `verify` gives the expected issues for every directory of every fixture; groups pass, other non-unit directories are `ignored`, `showcase: false` nodes have no unit issues.
3. TypeScript selection follows §3.1 (`react-single`, `ts-none`, `ts-7`).
4. `scaffold` produces the expected files and entry edit for each case of §7, never `overwrite` except the entry.
5. `ReactDefaultModule` is loadable (default export, `apiVersion`, schema) and passes the type test.
6. `npm run ci` passes, incl. this repo's packages following the unit convention.

## Recorded for later chunks

- **C9:** `check`/`init` with `react-default` over the C8 fixtures, end to end; `init` prints `ScaffoldResult.notes`; the hierarchy helper's amended rules (below).
- **C10:** finalise the showcase format and `ShowcaseProps`; read `shell` and `name` at runtime consistently with §5; implement `testRunner`, `viteConfig`, `showcaseStylesheets`, `showcaseSetup`; the layout module's own options; honour `showcase: false` (keep visible descendants reachable), `slug` in `resolveRoute` (URL-safe, unique), group pages with showcases, a unit at the unit root (whose README may be the package's and home's).
- **C13:** directory configs and claims; keeping showcase files and directory configs out of the published package; the kind heuristics and overrides; `showcase: false` switching off every rule; module-side memoisation and read tracking.
- **Later:** TypeScript 7's own API once stable (7.1).

## Amendments (proposed with this plan)

- **C2 B9:**
  - §5.3 `RepoInput.overrides: { directories: { [dir: RepoPath]: DirectoryOverrides }; kinds: { [declaration: string]: UnitKind } }`, `DirectoryOverrides { scope?: string[] /* repo-relative patterns */; showcase?: boolean; name?: string; slug?: string; order?: number; source: RepoPath /* where it was declared, for issues */ }`; the claim semantics (§4.3 of this plan) documented in `module/` as binding for modules.
  - §4.1 `UnitNode` and `GroupNode` gain `showcase: boolean` (default `true`; the `data/` README and JSDoc distinguish it from `showcases[]`, R2-2) and `order?: number`; `GroupNode` gains `showcases: ShowcaseEntry[]`; a directory config's `name` wins over the module's and the directory name. The CLI sorts siblings by `order`, then name.
  - §5.4 `PackageAnalysis.groups?: { dir: RepoPath; showcases: ShowcaseEntry[] }[]`; D70: units **at or below** `unitRoot`.
  - §5.5 `VerifyContext.group?: { readonly readme?: RepoPath }` when the CLI's hierarchy has a group at the directory.
  - §5.6 unit and group `RouteTarget`s gain `slug?: string` from the directory config; the module decides how to use it.
- **C4:** the scan collects `.tsshowcase` and `.tsshowcase.{json,jsonc,js,mjs,ts,mts}` files in scanned directories (several in one directory is an error); `-test` gains `createTestRepoInput()` (§10).
- **C6:** loads and validates directory configs (JSONC for the extensionless file, `.json`, `.jsonc`; jiti for JS/TS, default-exporting an object, D115), `directory-config.schema.json` with sidecar, `defineDirectoryConfig()`; the root and package configs gain `units` (`{ [dir]: DirectoryConfig }`, keys relative to the repo root or package) and `kinds` (`{ '<file>#<name>': kind }`); the same directory or declaration configured in two places is an error; everything normalised into `RepoInput.overrides`.
- **C9:** groups (D157): a non-unit directory at or below `unitRoot` with units below it, a README, showcases (`PackageAnalysis.groups`) or a directory config; `name`/`order`/`showcase` applied to groups and units; `VerifyContext.group` (the helper runs between `analyze` and `verify`); a warning for a directory config that applies to nothing or lies outside every `unitRoot`.
- **C10, C13:** the "Recorded" items above. **C11:** a changed directory config triggers re-analysis.
- **Super-plan R10** (C8 bullets: units by declaring directory and claims, directory configs, showcase format with `name`; C2/C4/C6/C9/C10/C11 bullets as above). **AGENTS:** D149–D159 (recorded), open points updated, C8 tracker.

## Open questions

- ✅ **Q1 Showcase file format (D149):** one showcase per file; `export const name` else the file name; `export const shell = true`; props typed however the author likes, only the shape matters.
- ✅ **Q2 Unit directories (D153, D154):** the declaring file's directory unless claimed; CLI-owned directory configs `.tsshowcase.*` claim downwards with `scope`, nearest explicit wins; the unit root can be a unit; `unitRoot` stays the entries' common directory, with `react/outside-unit-root`; same contents in config `units` maps; `name`, `slug`, `order` (D156).
- ✅ **Q3 `init` semantics (D150).**
- ✅ **Q4 Groups (D157):** any non-unit directory under the unit root with units below, a README, showcases or a directory config; groups may have showcases; `showcase: false` hides a node (D155).
- ✅ **Q5 Showcases out of the library build (D151):** an `init` note and a `check` warning.
- ✅ **Q6 TypeScript fallback (D152).**
- ✅ **Q7 Kind overrides (D158):** keyed by `'<file>#<declared name>'`, in config files and directory configs.
- ✅ **Q8 `headless` preset (D159).**
- ✅ **R2-1:** a directory config whose directory has no declarations (own or claimed) makes it a group, not an "empty unit" warning; the CLI warns only when it applies to nothing (§4.3, C9). Agreed; refines the "declared a unit but exports nothing" warning agreed earlier in round 1's discussion.
- ✅ **R2-2:** the data field keeps the config key's name, `UnitNode.showcase` / `GroupNode.showcase`; the `data/` README and JSDoc spell out the difference from `showcases[]` for module authors (the user: it only matters to module implementors, so documenting it is enough).
