# Chunk plan C8: React base, Node side (`@ofzza/tsshowcase-base-react`, `@ofzza/tsshowcase-module-react-default`)

> Status: **APPROVED** (round 3, 2026-10-22: the rewrite on the representation model, plan 11 of the redesign's application; approved together with C2 amendment B11 and the wording in C1, C4, C6, C15 and the super-plan). Rounds 1–2 (2026-10-19, units, claims and groups; decisions D149–D159) are in this file's git history and `08a-c8-plan-review.md`; the representation redesign (`08b-representation-redesign.md`, D160–D210) replaced their model, and this rewrite applies 08b §12 ("C8"), the AGENTS open points for C8 and the C8 items handed over by plans 3–10 (`01e`, `04b`, `05b`, `06b`, `15d`, `01h`, the approved plans' "Recorded for later chunks"). Round 2's proposed amendments (its B9 and R10) are void. The draft's choices Q1–Q9 and the review's questions QA–QF were settled with the user one by one (D211–D218); the two zero-context reviews' fixes (Fix 1–18) are applied. Review record: `08g-c8-rewrite-review.md`. Decisions referenced as `D<n>` are in AGENTS.md §3.

## Goal

Everything `check` and `init` need from a module for React/TypeScript repos, with no bundler involvement (D48):

- **`analyze`** (D170, D171, D200): package entries → public symbols attached to their declaring files, with kinds by value; showcasing components in showcase files (D195, D214); a few analysis issues (`react/*`), including `react/showcase-in-build` (D151);
- **`verify`** per file (D177): the showcase format checks and the opt-in **`headless`** preset (D47, D159);
- **`scaffold`** for `init` (D196, D215);
- the **preliminary React showcase file format** (D96, D195, D214) and its **`showcase()` helper**, finalised in C10;
- the first version of the **base module class** (`analyze`, `verify`, `scaffold`, `dispose`), plus the loadable **`react-default` class** extending it, so C9 can run `--module react-default` before C10 exists.

**Not C8's any more** (08b, D166, D172): the representation check (`represent/*`), associations, nodes, directory configs, kind overrides, exemptions and collections (the CLI, C6 and C9), test runners and test results (the CLI and C7). C10 adds `resolveRoute`, `build` and `dev` to the same classes.

**Depends on:** C2 (the contract, incl. amendment B11), C4 (the `-test` package, `copyFixture` with `linkNodeModules`, the fixture conventions).

## Requirements this plan answers

| Source | Point | Where |
| ------ | ----- | ----- |
| 08b §12 C8, D170, D171 | `AnalysisResult`: symbols attached per file, showcasing components with `meta`, `inputs`, `state` | §4 |
| D200, 08d V5, C2 §5.4 (B10) | kinds by value, framework kinds first; the syntactic fallback without `@types/react` | §4.3 (D213) |
| D152, 01b K19 | which TypeScript | §3.1 |
| 08a U9–U13 (carried) | entries, dist → source, externals, ignored sources, TS API precision | §3.2, §4.1, §4.2 |
| 08a U7 (carried) | namespace re-exports, `export =` | §4.2 (D211) |
| `01e`, `15d` G17 | ambient declarations; a declaration-only `exports` subpath; per-part tsconfigs; re-exports across workspace packages | §3.2 (D217), §4.1, §4.2 (D212) |
| `04b` (D201) | `.TS`/`.d.TS` sources, `config/same-base` with `Button.TSX` | §3.1 |
| D195, D96, `01e` | the showcase file format, `shell` and titles in `ShowcasingComponent.meta` | §4.4, §5 (D214) |
| D177, D159, D47 | per-file `verify`, the `headless` preset | §6 |
| C4 M11 | module-side memoisation defeats read tracking | §6.4 |
| D151 | `react/showcase-in-build` as an analysis issue | §4.5 |
| D196, 08b §10, 08a U8/U18 | `scaffold` | §7 (D215, D216) |
| D116, D138, D145, D148, D172, `06b` R2#27, `15d` | the options schema; no `testRunner`; kind keys; rules outside reserved namespaces | §2, §8 |
| C1 A3, C4 | fixtures (committed `dist/` re-included) | §9 |
| 08a U17, U20–U22 | `inputs`, refresh boundaries, root-set per-package options, test rationale | §4.6, §5, §8, §10 |

From 08a, U1–U6, U14, U16, U18's claim parts and U23 (except what 08b §12 hands to C6, C12, C13, C14) are obsolete; U15 (slug validation) went to C6 (CC10); U19 (showcase exports by syntax) is replaced by the `showcase()` helper (D214).

## 1. Where the code lives (dogfood layout, C1 §8)

```
packages/tsshowcase-base-react/src/
  index.ts                         entry '.': named re-exports only (C1 §8)
  index.test.ts                    wiring test
  module/
    README.md                      documents the exporting files in this directory
    ReactModuleBase.ts             class: the base module class (§2)                     ReactModuleBase.test.ts
    ReactModuleOptions.ts          type-only: the options type inferred from the schema
    REACT_RULES.ts                 constant: the RuleDescriptor list (§6.1)
    options.schema.json            + sidecar (D62); not exported (reached through ReactModuleBase.optionsSchema)
  showcase/
    index.ts                       entry './showcase' (browser-safe: imports nothing from Node or TypeScript)
    README.md
    showcase.ts                    function: the showcase() helper (§5)                 showcase.test.ts
    ShowcaseProps.ts               type-only: ShowcaseProps, ShowcaseSettings (§5)
  internal/                        no public symbols; tests per file (C1 §4, §10)
    typescript/                    loading TypeScript (§3.1), tsconfigs and programs (§3.2)
    analysis/                      entries (§4.1), the export walk (§4.2), kinds (§4.3), showcase files (§4.4), build leaks (§4.5)
    verify/                        showcase rules, headless rules (§6)
    scaffold/                      targets, stubs, templates, the entry edit (§7)

packages/tsshowcase-module-react-default/src/
  index.ts                         export { ReactDefaultModule, ReactDefaultModule as default } from './module/ReactDefaultModule.js'
  index.test.ts
  module/
    README.md
    ReactDefaultModule.ts          class: `class ReactDefaultModule extends ReactModuleBase` (C8: display name only; C10: layout, build, dev)
    ReactDefaultModule.test.ts
```

- `-base-react`'s `exports`: `.` and `./showcase`, each with a `source` condition (C1 §8), plus `./package.json`.
- Under C1 §8 and D162: the two classes and `showcase` need docs and tests; `ReactModuleOptions`, `ShowcaseProps`, `ShowcaseSettings` (types) and `REACT_RULES` (a constant) need docs only; nothing here is a component or hook, so no showcases (C10 adds the runtime components and their showcases).
- `react-default`'s package default-exports the class (D9) and also exports it by name; both re-export the one declaration, so it is one symbol (D171).
- **Dependencies of `-base-react` in C8:** `@ofzza/tsshowcase-api`, `@typescript/typescript6` (§3.1), `semver`, `picomatch` (§4.1, §4.5). No React and no Vite in C8 (C10 adds them; `-base-vite` is already declared by C1). The `./showcase` subpath needs no React types (§5).

## 2. The classes

```ts
export class ReactModuleBase implements ShowcaseModule<ReactModuleOptions> {
  static readonly apiVersion = '^0.1';
  static readonly displayName: string = 'react-base'; // layout modules override it
  static readonly optionsSchema = optionsSchema; // §8
  static readonly rules = REACT_RULES; // §6.1; namespaces react/ and headless/, never CLI-reserved ones (C2 §9)
  static readonly optionFlags = { kind: 'scaffold.kind', preset: 'presets' } as const; // D79, §8
  // no `kinds`: components and hooks are well-known kinds (D162); no `testRunners`: the built-in Vitest runner is a candidate anyway (D172)

  protected readonly options: ReactModuleOptions; // the root options (ModuleInit.options)
  protected readonly logger: Logger;
  constructor(init: ModuleInit<ReactModuleOptions>);

  analyze(ctx: AnalyzeContext<ReactModuleOptions>): Promise<AnalysisResult>; // §4
  verify(file: FileInput, ctx: VerifyContext<ReactModuleOptions>): Promise<readonly Issue[]>; // §6
  scaffold(ctx: ScaffoldContext<ReactModuleOptions>): Promise<ScaffoldResult>; // §7
  dispose(): void; // drops the per-instance TypeScript cache (§3.1)
}
ReactModuleBase satisfies ShowcaseModuleConstructor<ReactModuleOptions>;

export class ReactDefaultModule extends ReactModuleBase {
  static override readonly displayName = 'react-default';
}
```

- **Not loadable on its own** (super-plan): `-base-react` has no default export. `ReactModuleBase` is concrete, so layout modules extend it without boilerplate; C10 decides whether its layout hooks become abstract.
- Static members are inherited, so `ReactDefaultModule` above is a complete C8 module.
- **No per-call state on `this`** (C2 §5.1): `verify` runs concurrently. The only instance state is the cache of loaded TypeScript instances (§3.1) and the flags that keep once-per-run warnings to one (§3.1, §8); neither holds repo data.

## 3. TypeScript

### 3.1 Which TypeScript (D152)

Per package, `analyze` resolves `typescript` with `createRequire` from the package directory, then from the repo root (hoisted installs). The version is read from `typescript/package.json` without importing the package.

| Found | Used | Log |
| ----- | ---- | --- |
| `>=5.0 <7` | the repo's own | debug line with version and path |
| `>=7` | the bundled TS 6 API (`@typescript/typescript6`) | one warning per run: "TypeScript 7 has no stable programmatic API yet; analysing with TypeScript 6.0" |
| `<5.0` | the bundled TS 6 API | one warning per run: "TypeScript <x> is older than 5.0; analysing with TypeScript 6.0" |
| none | the bundled TS 6 API | one info line |

- **Facts (checked 2026-10-22):** `typescript@latest` is 7.0.2; its `exports` offer `.` (`lib/version.cjs`), `./package.json` and `unstable/*` APIs only; 7.1 is in beta (`7.1.0-beta`). `@typescript/typescript6@6.0.2` (the TypeScript team's) has `main` → the 6.x JS API and is a thin wrapper over `@typescript/old` (`npm:typescript@^6`), so the analysing version floats within 6.x (08a C26). A plain `typescript@6` dependency would be hoisted next to, or clash with, the repo's own `typescript` in the CLI's install tree; the scoped name never does.
- Loaded instances are cached per resolved path on the module instance; `dispose()` drops the cache.
- `verify` and the showcase-file parse (§4.4) use the bundled TS 6 for syntax only (`ts.createSourceFile`), never a program.
- TypeScript only treats lower-case extensions as sources, so `Button.TS` and `index.d.TS` yield no symbols (harmless; D201 governs the CLI's own classifications, `04b`). A `Button.TSX` next to `Button.tsx` is the CLI's `config/same-base` (D173).

### 3.2 Which tsconfig, and the programs (D217)

- **Per entry source**, the tsconfig that covers it: the `tsconfig` option (§8, per package) when set; else the first `tsconfig.json` found walking up from the entry source's directory to the package directory, and on to the repo root, whose parsed `fileNames` include the source. A **solution-style** tsconfig (`files: []` plus `references`, e.g. the Vite React template) is followed through its `references`, recursively, to the project that includes the source. With none, defaults: `allowJs`, `jsx: preserve`, `module: esnext`, `moduleResolution: bundler`, `resolveJsonModule`, `target: es2022`. Configs are read with `ts.getParsedCommandLineOfConfigFile` (comments, `extends`).
- So a package with **per-part tsconfigs** (a `runtime/` directory with its own `dom`/`jsx` options, C1 §4, C15 §1) analyses each part's entry under its own options (`01e`).
- **Root files:** the entry sources the config covers, plus the config's `.d.ts` `fileNames` inside the package and outside `node_modules` (ambient context such as `vite-env.d.ts`, `declare module '*.svg'`; R1#8). Ambient files are never exporting candidates (§4.2).
- **Overrides on top:** `noEmit`, `allowJs`, `skipLibCheck`, `resolveJsonModule`, and the `conditions` option (§8) prepended to `customConditions` when `moduleResolution` supports it (`node16`, `nodenext`, `bundler`).
- **One program per distinct (TypeScript instance, effective compiler options)** (U13). A symbol reached from entries analysed by different programs is classified in the program of the first such entry (entry order).
- **The emit config** (for dist → source, §4.1, and build leaks, §4.5): the package's `tsconfig.build.json` when present, else the entry's config; solution-style ones followed through `references` the same way. A typical package has a type-check `tsconfig.json` and an emit `tsconfig.build.json` (this repo, C1 §2); only the emitting config knows where the output goes.
- Diagnostics are not reported: analysis is not type-checking. Unresolved imports make types `any` (§4.3).

## 4. `analyze` (D170)

For every package in `ctx.repo.packages`, checking `ctx.signal` between packages and entries. The analysis reads entries, barrels and declarations from disk wherever they are (08b §3.1); only its *results* are limited: it returns files at or below the package directory that exist through `ctx.repo.files` (ignored = nonexistent, D103), and the CLI drops keys outside scope silently (D170).

### 4.1 Entries, and dist → source (U9, U10, D218)

1. **Entry list** from `package.json`:
   - `exports`: a string or a condition object without `.`-keys means `'.'`; otherwise every subpath key. Subpaths mapped to `null` are skipped. Subpaths whose resolved targets are neither scripts nor declaration files (`./package.json`, `./styles.css`, other assets, JSON) are skipped at debug level.
   - **Subpath patterns** (`'./*'`, `'./components/*'`) are **expanded** (D218): the target is chosen in condition order (step 2); when it points into an output directory, the dist → source rules (step 3) are applied to the pattern's prefix and suffix; the `*` is then expanded over the files that exist (ignored = nonexistent). Every match that is a script or declaration module becomes an entry; other matches (CSS, assets, `-theme-default`'s `"./css/*": "./css/*"`) are skipped silently.
   - Without `exports`: `types`/`typings`, `module`, `main`, in that order, as `'.'`.
   - Without any of these: `src/index.{ts,tsx,mts,js,jsx,mjs}`, then `index.{…}` at the package root.
   - The `entries` option (§8, per package, `{ subpath: path }`) replaces the list entirely.
2. **Target per subpath:** conditions in the order: the `conditions` option (default `['source']`), then `types`, `import`, `module`, `default`, `node`, `require`; nested condition objects recursively; arrays take their first string. Each matching condition is tried in that order until one maps to a source (U10).
3. **Map to source** (the emit config, §3.2):
   - a TypeScript source (`.ts`, `.tsx`, `.mts`, `.cts`) is used as is;
   - a declaration file (`.d.ts`, `.d.mts`, `.d.cts`) or a JS file under an **output directory** is mapped back: through `outDir` (and `declarationDir` for declarations) → `rootDir`, swapping the extension (`.js` → `.ts`, `.tsx`, and `.js`/`.jsx` for `allowJs` packages; `.jsx` → `.tsx`, `.jsx`; `.mjs` → `.mts`; `.cjs` → `.cts`; `.d.ts` → `.ts`, `.tsx`; `.d.mts` → `.mts`; `.d.cts` → `.cts`). When `rootDir` is unset, both `src/` (TypeScript 5's inferred common root) and the tsconfig's directory (TypeScript 6's default) are tried, keeping the candidate that exists (R1#12). Without an `outDir`, a leading `dist`/`build`/`lib`/`out`/`esm`/`cjs` segment maps to `src/`;
   - **output directories** are `outDir`, `declarationDir` and, without an `outDir`, the heuristic ones above; a **hand-written declaration file** is a declaration file outside them, used as is (U9; e.g. `-base-vite`'s `./client`, whose `source` condition points at `src/client/client.d.ts`, §4.2);
   - otherwise a JS file outside the output directories is used as is (JS packages, `allowJs`).
   - A candidate counts only if `ctx.repo.files.exists()` says so (ignored = nonexistent), so a gitignored `dist/` is never analysed.
4. A subpath with no usable source gets a **`react/entry-unresolved`** error on the package (hint: add a `source` export condition, or set `entries`). A package without any resolvable entry declares no symbols; it is still in the data (D193).

### 4.2 The export walk and attachment (D171, U7, U11–U13)

For each entry: `checker.getExportsOfModule(entryModuleSymbol)` gives the public names. Each is resolved with `getAliasedSymbol`, called only on alias symbols (it throws on others, 08a C8; members of `export *` come back as non-aliases), all the way down.

- **The declaring file:** the value declaration's file; for symbols without a value meaning, the first declaration outside a module augmentation (`declare module '…' { … }`, `declare global`). One symbol per (file, declared name); a symbol reachable from several entries or under several names appears once (exported alias names are deferred, 08b §13).
- **The declared name** is the declaration's identifier; `default` only when the declaration has none (`export default () => …`); a named default (`export default function Button`) is `Button` (08a C10: its symbol name is `default`, so the identifier is read from the declaration).
- **Type-only exports:** a symbol is exported type-only when every route to it is type-only: an alias with a `getTypeOnlyAliasDeclaration` (`export type { X }`, `import type`), or a member of an `export type * from …` / `export type * as ns from …` declaration (whose members aren't aliases, so the walk reads `ExportDeclaration.isTypeOnly` on the entry's and barrels' star exports; R1#4). **A value export anywhere wins** over a type-only one (D171's "the value meaning decides"): `export { Klass }` from one entry and `export type { Klass }` from another make `Klass` a `class`.
- **Declaration merging:** the value meaning decides the kind; `type` only without one (§4.3).
- **JSON modules:** a re-exported JSON module's default export (`export { default as configSchema } from './config.schema.json'`) attaches to the `.json` file, named `default` (the resolved symbol is named `export=`, so the name is set explicitly; R1#14), a `constant` (C1 §8). With `allowArbitraryExtensions` and a tsschema sidecar (C1 A1, this repo), TypeScript resolves the import to `config.schema.d.json.ts`: a declaration in an arbitrary-extension declaration file `<x>.d.<ext>.ts` attaches to `<x>.<ext>` when that file exists, named `default` (R2#1). Sidecars are never exporting files (C1 §8).
- **Namespace re-exports (D211, U7):** `export * as utils from './utils/index.js'`, and the equivalent `import * as utils from './utils/index.js'; export { utils }` (R1#13), make every member of the target module public as `utils.<member>`: each member attaches to its own declaring file exactly as an `export *` would; the namespace name itself is no symbol (its "declaration" is a module, not a value anyone declared). A guard stops recursive namespace re-exports. A namespace *declaration* (`export namespace X { … }`) is an ordinary symbol (§4.3).
- **`export =`** in an entry: a `react/export-equals` warning on the package ("CommonJS-style exports aren't analysed"); no symbols from that entry.
- **Ambient declarations (D212, `15d` G17, `01e`):** an entry source that is not a module (a declaration file without top-level `import`/`export`, holding only `declare module 'virtual:…' { … }` or globals, like `-base-vite`'s `src/client/client.d.ts` behind its types-only `./client` subpath) has no exports, so it yields **no symbols** and no issue (debug log). Ambient module declarations and global augmentations are never symbols, wherever they are: they describe other modules, not this package's exports.
- **Not attached** (D171; logged at debug level):
  - **external declarations (U11):** a file with a `node_modules` segment, outside the repo root, a TypeScript default library file, or outside the package (its deepest owning package is another one). So re-exports from npm dependencies and from sibling workspace packages are never symbols of the re-exporting package (C1 §8: public API from a dependency needs a local declaration);
  - **ignored files (U12):** a declaration in a file `ctx.repo.files.exists()` reports as missing (e.g. a gitignored `src/__generated__/`) gives a `react/ignored-source` warning on the package naming the file and the export (its symbols would otherwise go unchecked).
- **Order:** a file's symbols in declaration order (position in the file), as D178 needs.

### 4.3 Kinds (D200, 08d V5, D213)

Per symbol, from the checker, **in this order** (the first match wins):

| Kind | Rule |
| ---- | ---- |
| `type` | no value meaning, or exported type-only (§4.2): interfaces, type aliases, non-instantiated namespaces, `export type { X }` |
| `hook` | callable (call signatures on the value's type) and named `use` + an upper-case letter or digit (`useToggle`, `use3D`) |
| `component` | named with an upper-case first letter, not a React context (below), and one of: a call signature whose return type, or a member of its union, **is or extends React's `ReactElement` interface** (base types checked), which covers `JSX.Element`, `React.JSX.Element` (React 19 has no global `JSX`) and `ReactPortal`, matched **by symbol**, not by printed text (U13, R1#3); a value whose type has `$$typeof` (`memo`, `forwardRef`, `lazy`: `ExoticComponent`); a construct signature whose instance type derives from React's `Component` |
| `class` | constructible (construct signatures, abstract ones included): class declarations, `export const Foo = class {}` |
| `function` | callable: function declarations, overloads, `const f = () => …`, `const f = memoize(g)`, callable objects with properties |
| `constant` | any other value: objects, arrays, primitives, enums, instantiated namespaces, JSON modules, React contexts |

- **React contexts are not components** (R1#2): `createContext(…)` values (and their `Provider`/`Consumer`) have `$$typeof` and a call signature; a type with both `Provider` and `Consumer` members, or React's `Provider`/`Consumer` types themselves, is excluded from the component rule and falls through to `constant`.
- **The name used by the rules** is the declared name; for an anonymous default export (`default`), the declaring file's base (`Button.tsx` → `Button`), and for an `index.*` file its directory's name (`Button/index.tsx` → `Button`; 08d V5, R1#11).
- **Without React types** (`@types/react` not resolvable from the declaring file, so React's symbols can't be found): a **syntactic fallback** for the component rule: an upper-case name and a function or arrow whose body contains JSX, a call to `memo`/`forwardRef`/`lazy` imported from `react` (or `React.memo`, …), or a class extending `Component`/`PureComponent` from `react`. When the fallback classifies at least one symbol of a package as a component, the package gets one `react/approximate-kinds` warning ("kinds are approximate without `@types/react`; install it"). A pure-TS package uses no fallback and gets no warning.
- **Unresolved types:** a value whose type is `any` (an untyped import) has no signatures, so the hook and component rules can't match it; it is classified by its initializer's syntax only: a function or arrow → `function`, a class expression → `class`, anything else → `constant`. So `const f = memoize(g)` with an untyped `memoize` and a component wrapped by an untyped HOC are `constant`s (documented; a `kinds` override fixes them, D189).
- Exceptions are the user's: `kinds` overrides (D189) and `exempt` (D191), applied by the CLI. C8 emits only well-known kinds.

### 4.4 Showcase files and showcasing components (D195, D214)

- **Showcase files:** in-scope files of the package's tree (`ctx.tree`) named `*.showcase.tsx` or `*.showcase.jsx`, **matched exactly, lower-case** (a name the React module prescribes, like tsshowcase's own names under D201; Vite's React plugin wouldn't compile `.TSX` anyway; R2#10). They are parsed with `ts.createSourceFile` (syntax only, no program); they are not program root files.
- **Showcasing components** of a file, in source order with the default export first: **exactly the exports whose value is a call to the `showcase()` helper** (§5): `export const Sizes = showcase(SizesDemo, { title: 'Sizes' })`, `export default showcase(Demo)`, or a local `const X = showcase(…)` exported later (`export { X }`, `export { X as default }`). The call must use the binding imported from `@ofzza/tsshowcase-base-react/showcase` (an alias such as `import { showcase as s }` counts). Every other export (helpers, sub-components, constants, types) is ignored.
- **`meta`** (§5): `{ title?: string; shell?: true }`, from the call's second argument, only for what it sets.
- A showcase file with no showcasing component doesn't count (D195) and gets `react/showcase-empty` from `verify` (§6.1).
- Results go to `AnalysisResult.files[path].showcasingComponents`; a showcase file that is also an exporting file (re-exported from an entry) carries both.

### 4.5 `react/showcase-in-build` (D151)

Showcases (and directory configs) are never reachable from the entries, so entry-based bundlers skip them. Two leaks remain, checked per package when it has showcase files or directory config files in its tree:

- **`tsc`:** the emit config (§3.2: `tsconfig.build.json` when present, else the package's tsconfig when it emits; solution-style configs followed through `references`) compiles showcase files: a parsed `fileNames` includes one. Directory configs are dotfiles, which TypeScript's wildcard `include`s skip (spike, 2026-10-19), so they are never compiled unless named explicitly.
- **`npm pack`** (skipped for `private: true` packages, which are never packed; R1#10): `package.json#files` is absent, or its entries match a showcase file or a directory config file (entries are files, directories (everything below) or globs, matched with `picomatch`, `dot: true`; `!` entries exclude, as npm does; `.npmignore` is not evaluated: a documented approximation).

Either gives one **`react/showcase-in-build`** warning on the package, naming the tsconfig or `files` and the pattern to add (`exclude: ["**/*.showcase.*"]`, or narrowing `files`). `scaffold` adds the same text as a note when it creates a package's first showcase (§7). Nothing edits tsconfig files or `files`; bundler configs are not inspected (documented limit).

### 4.6 Result

- `files`: per declaring file its `symbols` (§4.2, §4.3), per showcase file its `showcasingComponents` (§4.4); no `meta` per file in C8.
- `issues`: the analysis issues (§6.1), each with `package` set and the file named in the message, no `path`: they concern a package's entries, manifest or build setup, which often lie outside scope, so they are placed on the package root (C2 §9).
- `inputs` (D68, U17): the programs' source files inside the repo root and outside `node_modules`, every package's `package.json`, every tsconfig read (each `extends` and `references` chain inside the repo, emit configs included), and the showcase files.
- `state`: plain data for `scaffold` (§7): per package its resolved entries (subpath → source file, whether it is a module), the specifier style of the entry files' relative re-exports, the module resolution of each entry's config, and the TypeScript used. Never a promise or a program (§6.4).
- `meta`: none in C8.
- Tests are not the module's concern (D172): the analysis neither lists nor maps test files. `AnalyzeContext.tree` carries only `doc` and `config` roles (D169); `scaffold` and `verify` see every role.

## 5. The preliminary React showcase file format and the `showcase()` helper (D96, D195, D214)

- **Files:** `*.showcase.tsx` / `*.showcase.jsx`, named after the file they cover (`Button.showcase.tsx` associates with `Button.tsx` by name, D173); a file may hold **several showcasing components**, identified by export name.
- **Each showcasing component is declared explicitly** with the helper from the browser-safe subpath `@ofzza/tsshowcase-base-react/showcase`:

  ```tsx
  import { showcase, type ShowcaseProps } from '@ofzza/tsshowcase-base-react/showcase';
  import { Button } from './Button.js';

  function SizesDemo({ colorScheme }: ShowcaseProps) {
    return <Button size="s" tone={colorScheme} />;
  }

  export const Sizes = showcase(SizesDemo, { title: 'Sizes and spacing' });
  export default showcase(() => <Button />);
  ```

  ```ts
  export interface ShowcaseProps {
    readonly colorScheme: 'light' | 'dark'; // the showcase's selected scheme (D89, D90)
  }

  export interface ShowcaseSettings {
    readonly title?: string; // display title; without it C10 derives one from the export name (the default export from the file name)
    readonly shell?: true; // render inside the shell (catalogue-styled components only, D41, D100): no scheme selector
  }

  /** Marks a component as a showcase. Returns the component unchanged: settings are read statically by the module, never at runtime. */
  export function showcase<TComponent>(component: TComponent, settings?: ShowcaseSettings): TComponent;
  ```

- **Settings are read syntactically** (§4.4): the second argument must be an object literal whose `title` is a string literal and whose `shell` is the literal `true`; anything else is `react/showcase-meta`. They become `ShowcasingComponent.meta` (`{ title, shell }`), which the CLI copies to `ShowcaseAssociation.meta` (C2 §8 rule 7: the React `shell` key).
- **The component** receives the showcase props; how the props are typed is the author's choice (`ShowcaseProps`, inline, or none); nothing checks them. Any component form works (`memo`, `forwardRef`, classes, annotated consts), since the helper wraps a value instead of decorating it.
- **Runtime:** `showcase()` returns its argument, so the copy of `-base-react` in the user's repo and the CLI's copy may differ without effect, and the file's exports stay components, so it remains a React Fast Refresh boundary (C15 S5; round 2's `export const name`/`shell` didn't, 08a U20). The subpath imports nothing from Node, TypeScript or the module class, so it is safe in the showcase bundle.
- **The dependency:** a repo with showcases declares `@ofzza/tsshowcase-base-react` as a devDependency of the showcase's package, or of the repo root in a workspace (hoisted installs); otherwise `verify` warns (`react/showcase-dependency`, §6.1), before `build` would fail to resolve the import.
- C10 finalises the format (and re-approves C8 if it changes).

## 6. `verify` (per file, D177)

Called by the CLI for every in-scope file; returns `[]` for files it has nothing to say about. Issues get the input file as `path` (the CLI forces it).

### 6.1 Rules

| Rule id | Default | Raised by | Check |
| ------- | ------- | --------- | ----- |
| `react/entry-unresolved` | error | analysis, package | §4.1 |
| `react/export-equals` | warning | analysis, package | §4.2 |
| `react/ignored-source` | warning | analysis, package | §4.2 |
| `react/approximate-kinds` | warning | analysis, package | §4.3 |
| `react/showcase-in-build` | warning | analysis, package | §4.5 |
| `react/showcase-empty` | error | `verify`, showcase-named file | a `*.showcase.{tsx,jsx}` file without a `showcase()` export (§4.4): it doesn't count as a showcase |
| `react/showcase-meta` | error | `verify`, showcase file | a `showcase()` call whose settings aren't an object literal of literal values (`title`: string, `shell`: `true`), or that has no component argument |
| `react/showcase-dependency` | warning | `verify`, showcase file | neither the file's package nor the repo root declares `@ofzza/tsshowcase-base-react` in `dependencies` or `devDependencies` (`PackageInfo.raw`) |
| `headless/files` | error | `verify`, component file, preset `headless` | for a component `X` declared in a file of its package: `use<X>.ts(x)`, `render<X>.tsx` and `<X>.types.ts` exist next to it |
| `headless/exports` | error | same | `use<X>` (a hook) and `render<X>` (a function) are public symbols of those files (from `ctx.representation`) |
| `headless/tests` | error | same | test files named `use<X>.test.ts(x)` and `<X>.test.tsx` exist next to it (matched by name in `ctx.tree`, not by the `test` role, which is missing when no runner listed tests; R2#9) |
| `headless/showcase-uses-hook` | warning | `verify`, showcase file, preset `headless` | for each component `X` of a node the file is a primary or strong showcase of, the file calls `use<X>(` (a textual match) |

- Removed with the redesign: `react/readme-required`, `react/test-required`, `react/showcase-required` (the CLI's `represent/*`, D174), `react/outside-unit-root`, `react/unused-claim`, `react/unused-kind`, `react/invalid-kind` (the CLI's `config/*`, D175, D189), `react/showcase-export`, `react/showcase-shell`, `react/showcase-name` (replaced by the helper, D214), `react/entry-pattern` (patterns are expanded, D218).
- `headless/tests` overlaps the CLI's test requirement only partly: the check accepts any test for a collection grouping the split, the preset asks for one per part (D159).
- Every issue carries a `hint` where a fix is obvious (`run \`tsshowcase init <path>\``, the pattern to add, the devDependency to declare). Severities are defaults; the CLI applies `rules` overrides (D176).

### 6.2 Presets

`presets: ['headless']` (§8, per package). Unknown presets are rejected by the schema's enum. The `headless` preset is documented in detail (§11, D159).

### 6.3 What `verify` reads

- `file` (`roles`, `symbols`, `showcasingComponents`, `associations`, `read()`), `ctx.tree` (siblings' existence and roles), `ctx.representation` (the items of the nodes a showcase belongs to, the public symbols of sibling files), `ctx.repo` (the root manifest's `PackageInfo`, the package's) and `ctx.package.options` (presets).
- Showcase checks re-parse the file from `file.read()` with `ts.createSourceFile` (syntax only, §3.1).

### 6.4 Read tracking (C4 M11)

`verify` reads file contents only through `file.read()` (and `ctx.repo.files`), so the CLI's tracking sees them; everything else it uses is plain data the CLI re-supplies when it changes (any change to the tree, enrichments or the representation re-verifies every file, D177). No promise is cached in `AnalysisResult.state` during `analyze` and awaited in `verify`.

## 7. `scaffold` for `init` (D196, D215, D216, 08b §10)

`ScaffoldContext` is built up to the representation (node issues empty, `tests.reason: 'init'`). The module decides what is missing from the nodes' docs, tests and showcases and each item's required resource types (`RepresentedItem.requires`, C2 B11), exactly as the check would (D174: a symbol is satisfied when its direct or collection node, or any configured node listing it, has a primary or strong resource of each required type), never from issues.

### 7.1 Targets (D215)

The CLI resolves `init <path>` (C9): it makes the cwd-relative path a `RepoPath`, rejects ignored targets and those outside the repo, and for a nonexistent path those outside every package or out of scope, then passes `ScaffoldContext.target: ScaffoldTarget { path; type: 'directory' | 'file' | 'new'; package? }` (C2 B11). The module never re-implements scope.

| `target.type` | `init` does |
| ------------- | ----------- |
| `directory` (any existing directory: in scope, a package directory, `.`) | for every direct and collection node in scope at or below it: the missing stubs (§7.2); an error when there is none ("no public symbols in scope below `<path>`") |
| `file`, an exporting file | the missing stubs of its direct (or collection) node |
| `file`, any other file | an error ("`<path>` declares no public symbol") |
| `new`, with a source extension (`.ts`, `.tsx`, `.mts`, `.js`, `.jsx`, `.mjs`) | a **new item**: that file (its directory created if needed), its stubs, and the entry export (§7.3) |
| `new`, any other path | a **new item in a new directory**: `<dir>/<Name>.<ext>`, `Name` the directory's base name, its stubs and the entry export |

- **New items:** the kind comes from `--kind` (`scaffold.kind`, D79), else from the name (`use*` → hook, upper-case first letter → component, else function). The extension: `.tsx` for components and hooks, `.ts` otherwise; `.jsx`/`.js` in packages whose entry sources are JavaScript. Their requirements come from `WELL_KNOWN_KINDS` (C2 B11, D216) for the kind; **exemptions are not applied** to new items (no symbol exists yet; documented: `init` never overwrites, an unwanted stub is deleted).
- **The `headless` preset**, for a new component `X`: `X.tsx`, `use<X>.ts`, `render<X>.tsx`, `<X>.types.ts`, each with the stubs its kind requires, and their exports; a note suggests grouping them with a `collections` entry (module-proposed collections are deferred, 08b §13).

### 7.2 Stubs

| Missing | Stub |
| ------- | ---- |
| docs | **file and new-file targets:** `<base>.md` (primary by name, covering only that file). **Directory and new-directory targets:** `README.md` in each directory below that has exporting files lacking docs and no README (it documents every exporting file there, strong, D173). **In a package or repo root directory, always `<base>.md`:** a root README is a home doc and takes no part in associations (E1). |
| tests | `<base>.test.tsx` when the node holds components or hooks, else `<base>.test.ts` (`.jsx`/`.js` in JS packages): `it.todo(…)` per symbol, importing from `vitest` (no testing library assumed; Vitest is the v1 runner, D172). **None** for symbols whose missing tests the check would suppress (no runner, a failed listing, or the file under a listing's `failedRoots`, from `representation.tests`; D172), with a note (R1#6) |
| showcase | `<base>.showcase.tsx` (`.jsx` in JS packages): one `showcase()` export per component or hook, rendering it (a hook through a small demo component calling it), with the props typed by `ShowcaseProps` (§5) |

- `<base>` is the exporting file's base, or the collection's key in the collection's directory (D190).
- **Requirements per existing item** come from `RepresentedItem.requires` (C2 B11): effective kinds after overrides, minus exemptions (D191). The module never reads configs.
- Templates: README (`# <directory name>`, one `##` section per exporting file listing its symbols, placeholder text), `<base>.md` (`# <base>`, its symbols), tests, showcases. No file header comments (the repo's conventions aren't known).

### 7.3 The entry export (U8, U18)

- **Which entry:** among the package's resolved entries (`state`, §4.6) whose source is a module (not a declaration file), the one whose source directory is the nearest ancestor of the new file; else `'.'`. A package with no usable entry: `scaffold` throws ("no entry found; add `exports`/`main`, or set `entries`") rather than inventing one (U8).
- **The edit:** a named re-export appended to that entry's source, following C1 §8's style: `export { Button } from './components/Button.js';` (`export type { … }` for types). The specifier copies the style of the file's existing relative re-exports (`.js`, `.ts` or extensionless), else `.js` under `node16`/`nodenext` and extensionless otherwise. `ifExists: 'overwrite'`, computed from the file's current contents (read through `ctx.repo.files`). Skipped with a note when the entry already exports the item.

### 7.4 Rules for every target

- **Never overwrites user files:** stubs are `ifExists: 'skip'`; only the entry edit is `overwrite`. Never writes directory configs, tsconfig files or `package.json`.
- `notes`: what was created and skipped; the build note (§4.5) when a package gets its first showcase; the devDependency to add when a showcase stub is written and `@ofzza/tsshowcase-base-react` isn't declared (§5); skipped test stubs (§7.2); for hooks the hint "a hook needs a showcase of its own; group it with its component in a collection, or `exempt` it" (`01e`).

## 8. Options (`module/options.schema.json`, D58, D62, D116, D148)

| Option | Type | Scope | Marker | Used by |
| ------ | ---- | ----- | ------ | ------- |
| `presets` | `('headless')[]`, default `[]` | per package | `x-tsshowcase-per-package` | verify, scaffold |
| `entries` | `{ [subpath]: string }` | per package | per-package; values `OPTION_FORMAT.path` | analyze |
| `tsconfig` | string | per package | per-package; `OPTION_FORMAT.path` | analyze |
| `conditions` | string[], default `['source']` | per package | per-package | analyze |
| `scaffold` | `{ kind?: 'component' \| 'hook' \| 'function' \| 'class' \| 'constant' \| 'type' }` | root only | – | scaffold; a flag in practice (`--kind`) |
| `viteConfig` | string | root only | `OPTION_FORMAT.path` | C10 (D138) |
| `showcaseStylesheets` | string[] | root only | items `OPTION_FORMAT.path` | C10 (D90) |
| `showcaseSetup` | string | root only | `OPTION_FORMAT.path` | C10 (D95) |

- **Root only by default (D148):** unmarked options say "root config only" in their `description`; per-package path values resolve against the package config's directory (D116).
- **No `testRunner` option** (D172): runners are CLI config (`testRunner`, `testRunners`); no migration hint (D208).
- **Root-set per-package options in a workspace (U21):** `entries` and `tsconfig` set in the root config apply to every package (D69, D148), which is almost never meant: `analyze` logs one warning per run when either is set in `ModuleInit.options` of a workspace.
- Kinds, exemptions, collections and directory configs are not module options: they are CLI config (D188–D191).
- C8 declares the C10 options now, so the schema, docs and dogfood config don't change shape later; C10 implements them. `homeDocs` is `react-default`'s own option and arrives with its layout and the design of how layout modules extend the schema (C10, D185).
- `optionFlags`: `--kind` → `scaffold.kind`, `--preset` → `presets` (D79).

## 9. Fixtures

New fixtures (own lockfile where they install packages, C1 §5). Fixture content this repo's `.gitignore` would catch (a committed `dist/`) is re-included with `!` rules (C1 A3); fixtures carry their own ignore rules in `.tsshowcaseignore` (D109).

| Fixture | Purpose |
| ------- | ------- |
| `react-single` | Single package, `typescript@5.x`, `react`, `@types/react@19`, `@ofzza/tsshowcase-base-react` as a devDependency (a `file:` link to this repo's package); `exports` with `source` and `dist` targets (no `dist/`); `scope: ['src']`; the C1 §8 layout in miniature: `src/components/` (`Button.tsx` with `Button.showcase.tsx` holding two `showcase()` exports, one with a `title`; `Card.tsx` with `Card.md`; a `README.md`), `src/hooks/useToggle.ts` (README, test, showcase), `src/utilities/format.ts` (overloads), `src/types/Theme.ts` (type-only), `src/constants/sizes.ts`, a JSON schema re-exported with a tsschema-style sidecar (`allowArbitraryExtensions`), `src/compositions/Forms.showcase.tsx` (a showcase of no exporting file, weak), a `.tsshowcase.jsonc` with a collection. The end-to-end fixture for C8 and C9. |
| `react-kinds`, `react-kinds-18` | Every form of §4.2–§4.3, under `@types/react@19` and `@types/react@18` (`JSX.Element` vs `React.JSX.Element`): function and arrow components, `React.FC`, `memo`/`forwardRef`/`lazy`, class components, contexts and providers (constants), PascalCase non-component functions, `use*` hooks (and a `use*` constant), anonymous and named default exports (incl. in `index.tsx`), aliases and several entries reaching one symbol, a value and a type-only export of one class, `export type *` and `export type * as ns`, declaration merging, overloads, callable objects, `memoize`d functions (typed and untyped), an untyped HOC, class expressions, abstract classes, enums, instantiated and type-only namespaces, `export * as ns` and `import * as; export { }`, a module augmentation, a JSON module |
| `react-no-types` | React-style sources without `@types/react`: the syntactic fallback and `react/approximate-kinds`; plus a pure-TS file (no warning from it) |
| `react-entries` | npm workspace, one package per entry case: `exports` string, condition object, subpaths incl. `null`, `./package.json`, `./styles.css`, `./css/*` (skipped silently), a script pattern `./components/*` and a dist-mapped pattern (expanded); `dist-only` (a committed `dist/`, ignored by the fixture's `.tsshowcaseignore`: asserts the mapping to `src/` through `tsconfig.build.json`'s `outDir`/`rootDir` and that `dist/` is never analysed); `dist-heuristic` (no `outDir`: `dist/` → `src/`); `rootdir-unset` (both candidates); `main-only`; `no-manifest-entry` (`src/index.ts` fallback); `unresolved`; `export-equals`; `ambient` (a types-only `./client` subpath to a non-module `.d.ts`); `option-entries` (the `entries` option in a package config) |
| `react-tsconfigs` | Per-part and solution-style tsconfigs (D217): a package whose `runtime/` part has its own tsconfig with `jsx: react-jsx` (its components classified correctly), a Vite-template-style `tsconfig.json` (`files: []`, `references` to `tsconfig.app.json`), an ambient `vite-env.d.ts` with `declare module '*.svg?react'` used by an export, a solution-style `tsconfig.build.json` leaking showcases |
| `react-workspace` | npm workspace: `packages/ui` (React; re-exports from `react` and the sibling `@fx/utils`: both external), `packages/utils` (pure TS), a package config with `presets`, a gitignored generated source re-exported (`react/ignored-source`), a nested package |
| `react-showcases` | The showcase format: several `showcase()` exports per file, an aliased helper import, `title`/`shell`, a `memo`, a `forwardRef` and a class component wrapped, invalid settings (`react/showcase-meta`), a showcase-named file without `showcase()` (`react/showcase-empty`), helper and sub-component exports (ignored), a `.jsx` showcase, a `.TSX` file (not a showcase), a showcase file that is also re-exported, a package without the devDependency (`react/showcase-dependency`) |
| `react-headless` | `presets: ['headless']`: one component following the split, one breaking each `headless/*` rule |
| `react-build-leak` | `tsconfig.build.json` compiling showcases, `files: ["src"]` publishing them and a `.tsshowcase.jsonc`, a `files` list with a `!` exclusion (no leak), a `private: true` package without `files` (no warning) |
| `react-scaffold` | `init` cases: a directory with missing stubs, a package directory under `scope: ['src']`, a collection, an existing file, new file and new directory targets, an exporting file directly in the package root (`<base>.md`), a `./hooks` subpath entry, a package without entry, each specifier style, an exemption, the `headless` preset, a repo without a test runner (no test stubs) |
| `ts-none` | No `typescript`, no tsconfig, plain `.js` sources: the bundled TS 6, `allowJs`. The TS 7 and TS < 5 rows of §3.1 reuse it with a fake `node_modules/typescript/package.json` written through `copyFixture`'s `files` (the version is read, never imported, so no install is needed) |

## 10. Tests

- **Contexts without the CLI (U22):** the scanner, the representation builder and the check are CLI-internal (C4, C9), and a published package shouldn't depend on CLI internals. The private `-test` package gains (C8; it then depends on `-api` for types and on `picomatch`):
  - `createTestRepoInput(dir, { packages?, ignored?, options? })`: an `fs`-backed `RepoInput` (`node_modules`/`.git` unreachable, `ignored` patterns treated as nonexistent, `PackageInput.options` and `scope` given by the test);
  - `createTestTree(repo, { roles? })`: a plain walk producing a `ScannedTree` in walk order, in-scope by each package's `scope`, with the `doc` and `config` roles by name (and any extra roles a test supplies for `verify`/`scaffold`);
  - small builders for `Representation`, `FileInput` and `ScaffoldTarget` values (nodes, items with `requires`, associations, `tests`), so `verify` and `scaffold` tests state exactly the representation they run against.
  The real pipeline over these fixtures is exercised end to end by C9 (`check` and `init --module react-default`).
- Per internal file: TypeScript selection (each row of §3.1), tsconfig selection per entry (part, solution-style, defaults) and the emit config, entry resolution (every case of §4.1, incl. patterns and the ignored `dist/`), the export walk (attachment, names, type-only routes, merging, JSON and sidecars, namespaces, `export =`, ambient declarations, externals, ignored sources, order), kinds (every row and fallback of §4.3), showcase files (§4.4), build leaks (§4.5), `inputs`, each rule of §6 positive and negative, scaffold planning for each case of §7.
- `analyze` over every fixture against an expected `AnalysisResult` (files, symbols with kinds, showcasing components with `meta`, issues).
- `showcase()` returns its argument unchanged; the `./showcase` subpath imports nothing outside itself (a test over its module graph).
- Read tracking: `verify` inside a tracking stub reads file contents only through `file.read()`.
- Type tests: `ReactModuleBase` and `ReactDefaultModule` satisfy `ShowcaseModuleConstructor<ReactModuleOptions>`; `ReactModuleOptions` infers the table of §8; `showcase()` preserves its argument's type.
- Fixture tests use `copyFixture({ linkNodeModules: true })` wherever resolution matters (§3.1, `@types/react`, externals, the devDependency): run in place, a fixture would find this repo's own `typescript` and `@types/react` (C1 §5).

## 11. Documentation

- `-base-react` README: what the package is (not loadable), the export list (both subpaths), the rules table, the showcase file format and `showcase()` (preliminary), the options table, the TypeScript and tsconfig selection, the analysis rules (entries, patterns and their mapping, attachment, type-only routes, kinds and their fallbacks, contexts, namespaces, ambient declarations, JSON and sidecars, externals) and their limits (untyped values and HOCs, `.npmignore` not evaluated, bundler configs not inspected).
- **The `headless` preset, in detail (D159):** what it is (the `useX` / `renderX` / `X` split and its files), why a library would want it, how to turn it on, what each `headless/*` rule checks, how to group the split with a collection, and **what can't be verified**: that `renderX` is pure, that `X` only wires `useX` into `renderX`, that the hook holds all the logic, or that a showcase calling `useX(` really builds a custom presentation (a textual match only).
- `-module-react-default` README: the built-in name; "C8: `check` and `init` only; `build`/`dev` arrive with C10" (removed by C10).
- Area READMEs per C1 §8 and D56. CONTRIBUTE.md glossary: external re-export, showcasing component, syntactic fallback. Fixtures README: the table above.

## Acceptance criteria

1. Over every fixture, `analyze` returns the expected `AnalysisResult`: symbols attached to their declaring files with their kinds, showcasing components with `meta`, the analysis issues and `inputs`; the result passes the CLI's validation (D170: no nonexistent or duplicate keys, only well-known kinds).
2. `verify` gives the expected issues for every in-scope file of the showcase and headless fixtures, and none elsewhere.
3. TypeScript selection follows §3.1 (`react-single`, `ts-none` and its faked variants); tsconfig selection follows §3.2 (`react-tsconfigs`).
4. `scaffold` produces the expected files and entry edit for each case of §7, never `overwrite` except the entry, and respects `requires`.
5. `ReactDefaultModule` is loadable (default export, `apiVersion`, schema) and passes the type tests.
6. This repo's packages follow C1 §8 and analyse as intended: e.g. `-base-vite`'s `./client` yields no symbols, `-theme-default`'s `./css/*` is skipped silently, the CLI's re-exported JSON schemas attach to their `.json` files (not the sidecars); `npm run ci` passes.

## Recorded for later chunks

- **C9:** `check` and `init --module react-default` end to end over the C8 fixtures; resolving and validating the `init <path>` target into `ScaffoldTarget` (D215); filling `RepresentedItem.requires` and using `WELL_KNOWN_KINDS` in the check (B11, D216); `init` prints `ScaffoldResult.notes`.
- **C10:** finalise the showcase format, `showcase()` and `ShowcaseProps`; read `title`/`shell` from `ShowcaseAssociation.meta`; display titles from export and file names; `-base-react`'s own showcases import `@ofzza/tsshowcase-base-react/showcase` (a self-reference by package name) and the runtime confirms Fast Refresh with `showcase()` exports; `homeDocs` and extending the base schema; implement `viteConfig`, `showcaseStylesheets`, `showcaseSetup`; display name defaults (`index.*` → its directory's name, 08d V5); presenting `requires` (e.g. "not required").
- **C12:** `init` over a copied fixture followed by a passing `check`; the showcase format end to end.
- **C13:** the kind rules and their limits (untyped values and HOCs, the fallback, contexts); namespaces, ambient declarations, patterns; the showcase format, `showcase()` and the devDependency; keeping showcase files and directory configs out of the published package; module-side memoisation and read tracking (C4 M11); `init` targets and stubs (new items ignore exemptions).
- **Later:** TypeScript 7's own API once stable (7.1); reusing programs across `dev` re-analyses; module-proposed collections for the `headless` split (08b §13); machine-readable locations for package-level analysis issues (`08g` Q7 (b)).

## Amendments (proposed with this plan)

- **C2 B11** (additive):
  - §4 `RepresentedItem.requires: readonly ResourceType[]` with `type ResourceType = 'docs' | 'tests' | 'showcase' | (string & {})`: the resource types the symbol requires after kind overrides, module kinds and exemptions, in the order docs, tests, showcase; set by the CLI wherever a `RepresentedItem` appears (`FileInput.symbols` included) (D216).
  - `-api` exports **`WELL_KNOWN_KINDS: { readonly [kind: string]: KindDescriptor }`**, the D162 table (`module/`), used by the CLI's check, by modules for new items and by docs (D216).
  - §5.10 `ScaffoldContext.target: ScaffoldTarget { path: RepoPath; type: 'directory' | 'file' | 'new'; package?: RepoPath }` (was a `RepoPath`), resolved and validated by the CLI (D215); "the directory passed to `init <dir>`" → "the path passed to `init <path>`".
- **C1** (wording): §8 "Entry files only re-export, by name" applies to module entries; a types-only subpath whose source is a non-module declaration file (ambient `declare module` declarations, e.g. `-base-vite`'s `./client`) declares rather than re-exports and yields no symbols (D212); "every `exports` subpath carries a `source` condition" → every subpath resolving to scripts or declarations (D218; `./package.json`, JSON schemas and `./css/*` need none); §2 dependency table: `-test` depends on `-api` (types) and `picomatch`.
- **C4** (wording): §11.1, `-test` gains `createTestRepoInput`, `createTestTree` and the builders (C8 §10), and depends on `-api` and `picomatch`.
- **C6** (wording): the `init` positional is `<path>` (§2, §3, its registry entry and C9 hand-offs).
- **C15** (wording): its "Recorded for later chunks" C8 item (`15d` G17) answered by D212; its C10 item's "a `shell` flag, if C8's format keeps one" → showcases are `showcase()` exports, settings never exports (D214).
- **Super-plan** (wording, per `01h`): the C8 stub replaced by this plan's goal, requirements and dependencies; C9's text `init <path>` and the target resolution; the CLI-duties table gains rows "`RepresentedItem.requires` per item (B11)" and "resolving and validating the `init` target" (both C9); noted in its status line.
- **AGENTS:** D211–D218; D167's "required positionals stay (`init <dir>`)" → `init <path>`; D196 → "refined by D215, D216"; D195 → "refined by D214"; D149 → "replaced by D214" is already "changed by D195" (kept, plus a note); the C8 open points removed (resolved here), the C9, C10 and C13 rows gaining this plan's hand-offs; the tracker row; 08b's "applied" banner and the "Current focus" section removed once this plan is approved (08e E6).

## Open questions

All settled with the user one by one (2026-10-22); details in `08g-c8-rewrite-review.md`.

- ✅ **Q1 Namespace re-exports (D211):** members attached individually, no symbol for the namespace (§4.2).
- ✅ **Q2 Ambient declarations (D212):** never symbols, no issue; C1 §8 wording for declaration-only subpaths (§4.2).
- ✅ **Q3 Kind details (D213):** hooks by callability and name; an anonymous default export named after its file (its directory for `index.*`); untyped values by syntax; `react/approximate-kinds` only when the fallback was used (§4.3).
- ✅ **Q4 + QA Showcase format (D214):** showcasing components are exactly the `showcase()` exports; settings in the call, read syntactically; the helper in a browser-safe subpath, the user's devDependency checked (§4.4, §5). QA revised Q4's static properties (they don't type-check on `memo`/`forwardRef`/class components; the user preferred the helper's explicitness).
- ✅ **Q5 + QB + QD `scaffold` targets (D215):** `init <path>` resolved by the CLI into `ScaffoldTarget`; directory and file targets, existing or new; docs stubs by target type, `<base>.md` at roots; the entry by nearest source directory (§7).
- ✅ **Q6 + QC Requirements (D216, C2 B11):** `RepresentedItem.requires` (an open union, fixed order) and `WELL_KNOWN_KINDS`; new items get every stub their kind requires (§7).
- ✅ **Q7** Analysis issues on the package, the file named in the message (§4.6).
- ✅ **Q8** Tests through `-test` helpers and hand-built representations; C9 end to end (§10).
- ✅ **Q9** `homeDocs` with C10; the base declares C10's options now (§8).
- ✅ **QE Tsconfig per entry (D217):** the nearest config covering the entry source, solution-style configs followed, ambient `.d.ts` files as context (§3.2).
- ✅ **QF Subpath patterns (D218):** expanded against existing files; non-script matches skipped silently; `react/entry-pattern` removed (§4.1).
