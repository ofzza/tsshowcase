# Chunk plan C6: Plugin loading + configuration (CLI core)

> Status: **APPROVED** (round 4, 2026-10-19). Open questions Q1–Q5 resolved (D111–D115); the fresh-eyes review `06a-c6-plan-review.md` (P1–P14) applied (D116, D117). Approved together with C2 amendment B5 and super-plan R6, both applied. **C6 amendment (C5 planning) approved 2026-10-19 with the C5 plan:** `docs/*` rule ids (§12). **C6 amendment (C7 planning) approved 2026-10-19 with the C7 plan:** §7 `testRunner` in package configs warned and dropped (D136); §11 runners receive `ctx.repo` (C2 B7). **C6 amendment (C15 planning) approved 2026-10-19 with the C15 plan:** §7 module options are root-only by default; only those the module's schema marks with `x-tsshowcase-per-package` are merged from package configs (D145, D148; replaces the hard-coded `testRunner` rule). Decisions referenced as `D<n>` are in AGENTS.md §3. **Wording from plan 5 (D201, `04b-c4-redesign-review.md`):** §4 config names match in exact case, extension included (tsshowcase's own file names are an exception to D201). **Redesign amendment (the representation redesign, `08b-representation-redesign.md`, D160–D208; plan 7 of its application) approved 2026-10-21** (choices CC1–CC10, review questions QA–QJ and Fix 1–25 in `06b-c6-redesign-review.md`; D206–D208): the config keys and schemas of the redesign (package-relative `scope`, `directories`, `kinds`, `exempt`, `strict`, `testRunner`/`testRunners` as CLI keys, `rules` with patterns, `vcs.ref`; `check`, `failOnTestFailure`, `units` removed), `directory-config.schema.json` and `defineDirectoryConfig`; the flags `--strict[=part]`, `--no-strict`, `--rule`, `--out`, `--no-tests`, `--vcs-ref`; the directory-config loader after the scan; packages that don't exist never configured; the CLI's own test-runner loading (D172) replacing `loadTestRunner`; module statics `kinds`, `testRunners`, reserved rule namespaces; the rule-id table (D175); see "Redesign amendment" below.

## Goal

Turn the command line, the root config file and the per-package config files into one **resolved configuration**, and load the plugins it names: the rendering module (instantiated, with validated options), theme sets (resolved and version-checked; C11 selects and scopes, K14) and the **test runners**, which the CLI owns since the redesign (D172: candidates, pinning, detection). Produce C4's scanner inputs from the config; after the scan, load the **directory configs** C4 found (D188, `04b` J8). Ship the user-facing config schemas, their inferred types and `defineConfig`, `definePackageConfig`, `defineDirectoryConfig` (D66).

C6 does not run any action. C9 (`check`, `init`) and C11 (`build`, `dev`) call it.

## 1. Where the code lives

```
packages/tsshowcase/src/
  config-schema/    exported area: config.schema.json, package-config.schema.json, directory-config.schema.json (+ tsschema sidecars, D62), configSchema, packageConfigSchema, directoryConfigSchema, TsshowcaseConfig, PackageConfig, DirectoryConfig, StrictPart
  define-config/    exported area: defineConfig(), definePackageConfig() (P6), defineDirectoryConfig() (redesign)
  args/             internal: flag tables, the argv normaliser (`--strict`, `--no-strict`, `--rule`), the two-phase argv parse (§3)
  config/           internal: discovery, loading, validation, error formatting, per-source config values, scanner inputs (§4–§7, §12)
  directory-configs/ internal: loadDirectoryConfigs() after the scan (§4.1, redesign)
  plugins/          internal: reference resolution (D9), jiti loading, version checks, module / theme / runner loading, test-runner candidates and detection (§8–§11)
  resolve-config/   internal: resolveConfig(), ties it together (§2)
```

The exported areas follow this repo's dogfood layout (C1 §8, A4): one exporting file per concept, one README per area, tests named after the files they cover. The internal directories follow D10 and have tests, like C4's `scan/`.

New runtime dependencies of the CLI package: `jiti` (2.x: TS/JS loading, aliases, ESM resolution), `ajv` (draft 2020-12), `semver`, `jsonc-parser` (JSON configs with comments, Q4), `@ofzza/tsschema` (types only, for the inferred config types; C1 A1).

## 2. The pipeline and the seam with C4, C9, C11

```ts
/** Called once per CLI invocation by every action (C9, C11), after the action and its own flags are known. */
export function resolveConfig(
  input: ResolveConfigInput,
): Promise<ResolvedConfig>;

export interface ResolveConfigInput {
  readonly action: "check" | "init" | "build" | "dev";
  readonly argv: readonly string[]; // everything after the action name
  readonly cwd: string; // process.cwd()
  readonly logger: Logger;
}
```

Every action's flags and positionals live in **one static registry** in `args/` (§3, P3): the CLI's own option flags (C6), including the config-backed `build --out`, `build`/`dev --no-tests` and `dev --host`/`--port` (D111: flags that map onto config keys are C6's), plus the action-only flags and positionals that C9 adds to it (`check --format`, `init <dir>`). Actions take **no optional positionals** (D167). `resolveConfig` reads the current action's entries from it; collisions are checked against all of it.

Order inside `resolveConfig`:

1. **Argv normaliser, then phase-1 argv parse** (§3): CLI flags only; module flags are kept as unparsed tokens.
2. **Root:** `--cwd`, else the process cwd; no upward search (C4 §3, L4). Must be an existing directory.
3. **Root config:** discover or take `--config` (§4), load, validate against `config.schema.json` (a `skip` key is held back for step 4).
4. **`detectWorkspace(root)`** (C4 §2); then the L4 check for a `skip` key in the root config (§4, P12); then **`outDir`** resolved (`--out` > config > default, D167) and an explicit one validated early (M13, §12), since step 5 needs it.
5. **Per-package configs** (§7), in the walk order of `WorkspaceInfo.packages` (C4: parents before nested packages), **only for packages that exist**: C4's `packageExists()` with the root `ignore` (config and `--ignore`), the explicit `outDir` (`--out` or config) and the skipped packages known so far (`04b` Q3); validated against `package-config.schema.json`.
6. **Module:** resolve the reference (flag → config → `react-default`), load, version-check, check its statics (§8, §9; `kinds`, `testRunners`, `rules`). Not yet instantiated.
7. **Phase-2 argv parse** with the module's `optionFlags` added (§3, §9.2); any unknown flag or extra positional is now an error.
8. **Options** (for the root and for exactly the packages that exist and aren't skipped, step 5): path-marked values in each source made absolute against that source's base (§9.4, D116); then root `moduleOptions` + flags → validate → `ModuleInit.options`; per package: root `moduleOptions` shallow-merged with the package's own marked keys + flags → validate → `PackageInput.options` (D69, D148).
9. **Scan inputs:** each package's effective `scope`, for exactly the packages that exist and aren't skipped (C4's `ScanOptions.scope` is total, `04b` Fix 3) (flag > package config > root config, validated per source with C4's `validateScope()` and handed over as written, `04b` Q2, QI), `ignore`, `skippedPackages`, `outDir` (resolved in step 4; only an explicit one goes to the scanner, P2), `vcs` overrides incl. `ref` (§12).
10. **Rules, kinds, exemptions, directories:** `rules` from every source kept per source (flags, each package config, the root config; D176), keys checked against the CLI's table (D175) and the module's rules (§12); `kinds`, `exempt` and `directories` from the root and package configs kept per source, keys as written, with the source's base directory (§7, CC4); their keys validated (§5, CC9); kind values (top-level `kinds` and the `kinds` inside `directories` entries) checked against the well-known kinds and the module's (D189, a `ConfigError`).
11. **Strict and tests:** `strict` (flags replace config; D164) and `tests` (`--no-tests`, D197) resolved.
12. **Instantiate the module** with `ModuleInit` (§9.3).

Themes are loaded on demand (`resolved.loadThemes()`, §10): only `build` and `dev` need them, and a broken theme reference must not fail `check`. **Test runners** are loaded on demand too (`resolved.loadTestRunners(repo)`, §11, CC3), after the scan, because `detect` needs the manifests; every action calls it once: `check` and `init` list test files (associations need the `test` role, D172; C2 §5.9), `build` and `dev` list and run them.

After `resolveConfig` the action calls `scanRepo(resolved.scan)` and `toRepoInput(scan, resolved.packageOptions)` (C4 §2), then **`loadDirectoryConfigs(scan, resolved)`** (§4.1, `04b` J8), then hands everything to C9.

```ts
export interface ResolvedConfig {
  readonly root: string; // absolute
  readonly configFile?: string; // absolute; undefined when none was found
  readonly workspace: WorkspaceInfo; // C4
  readonly module: LoadedModule; // §9: reference, resolved URL, constructor, instance
  readonly moduleOptions: unknown; // validated root options (ModuleInit.options)
  readonly packageOptions: ReadonlyMap<RepoPath, unknown>; // validated per-package options, for toRepoInput (D69)
  readonly rules: ResolvedRules; // per source: flags (`--rule`), each package config, the root config; with patterns (D176); C9 applies them
  readonly configMaps: ConfigMaps; // `directories`, `kinds`, `exempt` per source (root config, each package config), keys as written, each source with its base directory ('.' or the package); C9 merges (D188, D189, D191; CC4)
  readonly scan: Omit<ScanOptions, "logger">; // C4 ScanOptions (scope per package); the action adds its logger
  readonly themeReferences: readonly ThemeReference[]; // §10, already parsed; loaded by loadThemes()
  readonly defaultTheme?: string;
  readonly outDir: { readonly path: string; readonly explicit: boolean }; // absolute; `--out` (relative to the cwd) > config `outDir` (relative to the config base) > ./showcase relative to the cwd (D167, §6, P2); only `explicit` ones reach the scanner (D101)
  readonly strict: readonly StrictPart[]; // D164: the selected parts after flags and config; [] = none
  readonly tests: boolean; // false with --no-tests or config `tests: false` (D197)
  readonly dev: { readonly host?: string; readonly port?: number }; // K8
  readonly actionFlags: { readonly [name: string]: unknown }; // parsed values of the action-only flags in the registry (e.g. `format`)
  readonly positionals: { readonly [name: string]: string | undefined }; // by registry name, e.g. { dir: 'src/Button' } for `init <dir>`
  loadThemes(): Promise<readonly ResolvedTheme[]>; // §10; `ResolvedTheme` is the CLI's own type (moved out of -api, 02b Skip 2)
  loadTestRunners(repo: RunnerRepoInput): Promise<LoadedRunners>; // §11 (D172, CC3)
  pluginHost(name: string): PluginHost; // for any plugin the CLI constructs (prefixed logger, versions)
}

type RuleMap = { readonly [idOrPattern: string]: "error" | "warning" | "off" };

/** D176: kept per source; C9 consults flags > the issue's package config > root config, the first source with a match deciding. */
export interface ResolvedRules {
  readonly flags: RuleMap; // --rule, in argv order
  readonly packages: ReadonlyMap<RepoPath, { readonly file: string; readonly rules: RuleMap }>; // per package config that sets `rules`
  readonly root?: { readonly file: string; readonly rules: RuleMap };
}

/** One config source with the base its keys are relative to (CC4); the same form as a loaded directory config. */
export interface ConfigSource<T> {
  readonly file: string; // the config file it came from (for messages)
  readonly base: RepoPath; // '.' for the root config, the package directory for a package config, the directory for a directory config
  readonly value: T;
}

/** `directories`, `kinds`, `exempt` per source, keys as written (CC4); C9 merges them with the directory configs (D188, D189, D191). */
export interface ConfigMaps {
  readonly root?: ConfigSource<Pick<TsshowcaseConfig, "directories" | "kinds" | "exempt">>;
  readonly packages: ReadonlyMap<RepoPath, ConfigSource<Pick<PackageConfig, "directories" | "kinds" | "exempt">>>;
}
```

## 3. Argv parsing (Q1)

**Ownership (D111):** C6 owns the **flag machinery and every option-like flag** (the ones that map onto config keys, plus module flags, D79) and parses positionals; C9 owns the command framework: action dispatch, `--help`/`--version` rendering, the report. C9 declares its action-only flags (`--format`) and positionals (`init <dir>`) as entries in C6's static registry (§2); every config-backed flag, action-restricted ones included (`--out`, `--no-tests`, `--host`, `--port`), is C6's, so there is one parser and one collision check. Actions never take optional positionals (D167).

- **Parser:** `node:util` `parseArgs` (no dependency), in **two phases**, because module flags are only known once the module is loaded (D79) and the module reference may itself come from `--module` or from the config: phase 1 parses the CLI's flags with `strict: false` and `tokens: true`, keeping unknown tokens; phase 2 parses again with the module's flags added, `strict: true`, `allowPositionals: true`, and the action's positional arity checked (P2).
- **Registry entries:** flags `{ name, type: 'boolean' | 'string' | 'number', multiple?, negatable?, description, configKey?, actions }` and positionals `{ name, action, required, description, configKey? }`. `negatable` booleans accept `--no-<name>`; numbers are coerced and validated; `multiple` flags repeat (`--scope a --scope b`).
- **The argv normaliser (redesign, D164, D176):** runs before both phases, on the tokens before `--` only. It rewrites a bare `--strict` into the four explicit tokens `--strict=check --strict=tests --strict=unavailable --strict=unhandled`, so a following word stays a positional (and gets the hint below) instead of being taken as a value. `--strict=<part>` (only the `=` form selects parts, repeatable) adds one part. **The flags apply in command-line order (QB):** `--no-strict` clears the set, `--strict`/`--strict=<part>` add to it, so `--strict --no-strict` selects none and `--no-strict --strict=warnings` selects `warnings` (a shell alias can be overridden by appending); any `--strict`/`--no-strict` replaces config `strict`. Empty or unknown part names are rejected after parsing, naming the valid parts. `--rule <id>=<severity>` (repeatable) is split on the **last** `=` (an id starting with `-` uses `--rule=<id>=<severity>`); severities `error | warning | off`; ids may be patterns (`represent/*`); the same id given twice: the later value wins (QB). A stray positional where none is allowed gets a hint: `build --strict tests` → "`build` takes no positional arguments; did you mean `--strict=tests`?"; `build out` → "did you mean `--out out`?" (D167).
- **Help (P3):** `describeFlags(action, argv, cwd)` returns the effective list (CLI, action and module flags and positionals, each with its description) for C9's `--help`. It never throws: it loads the config and the module only as far as needed for the module's `optionFlags` (no options validation, no instantiation), and when that fails it returns the CLI's own entries plus a warning saying why the module's flags are missing. `--help` short-circuits `resolveConfig`.
- **Collisions (D79, P3):** a module flag is checked against **every** entry of the registry, for all actions (module flags are accepted on every action, so a clash is the same error whichever action runs), and against `help` and `version`. A module flag may not start with `no-`; the `no-` form of a negatable flag is reserved with it. The CLI also reserves the `vcs-` prefix. A clash is a load-time error naming the module.

## 4. Config files

- **Names (Q4):** `tsshowcase.config.ts`, `.mts`, `.js`, `.mjs`, `.json`, `.jsonc`, exact case, extension included (names tsshowcase prescribes are an exception to D201; `tsshowcase.config.TS` is an unrelated file) (comments allowed in both, below; D115), looked up in the root only (no upward search, L4). **Several present is an error** naming them (no precedence to guess). None present is fine: everything has a default.
- **`--config <path>`** (relative to the process cwd) loads that file instead; it must exist. Its directory becomes the **config base** for config-relative values (§6). The root stays `--cwd` / the process cwd.
- **Loading:** `.json` and `.jsonc` are parsed the same way (D115): `//` and `/* */` comments are allowed and ignored (parsed with Microsoft's `jsonc-parser`, the parser VS Code uses; syntax errors are reported with line and column); everything else is strict JSON, **trailing commas rejected**; `$schema` is allowed. `.jsonc` gets comment highlighting in editors out of the box; for a `.json` file with comments the CLI README shows the VS Code `files.associations` line; `.ts`/`.mts`/`.js`/`.mjs` through jiti (§8.3), taking the **default export**, which must be a plain object (or a promise of one), typically wrapped in `defineConfig` (D66). A default export that is a function is rejected with a clear error: every action needs the same settings to work together, so there is no per-action config (D115; it can be added later, D13).
- **Aliases for config files (L12, D117):** the same aliases as for path-loaded plugins (§8.3): every `@ofzza/tsshowcase*` package resolves to the running CLI's own copy, so `import { defineConfig } from '@ofzza/tsshowcase'` works under `npx` and pnpm without a local install.
- **Validation:** ajv against `config.schema.json` (§5), `allErrors: true`, **no `useDefaults`** for the config: defaults are applied in code, because "explicitly configured" matters (an explicit `outDir` is a hard ignore, D101, while the default `./showcase` is not).
- **A package config at the root (L4, P12):** checked after `detectWorkspace` (§2 step 4). A root config containing `skip` gets a dedicated error: "`skip` is only valid in a workspace package's config"; when C4 detected that the root is a package of a parent workspace, the hint adds "run tsshowcase from the workspace root at `<path>`".
- **Security note (documented, C13):** `.ts`/`.js` configs, directory configs and path-loaded plugins execute code; running `check` on an untrusted PR runs that code.

### 4.1 Directory configs (redesign, D188; `04b` J8)

`loadDirectoryConfigs(scan, resolved)` runs **after** the scan, over the files C4 found in scope (`RepoScan.directoryConfigs`, exact names, C4 §8.1):

```ts
export function loadDirectoryConfigs(scan: RepoScan, resolved: ResolvedConfig): Promise<LoadedDirectoryConfigs>;

export interface LoadedDirectoryConfigs {
  readonly configs: ReadonlyMap<RepoPath, ConfigSource<DirectoryConfig>>; // per directory; base = the directory; `true` normalised to {}
  readonly issues: readonly Issue[]; // config/several-configs (CC1)
}
```


- **Several files in one directory:** none of them is loaded; one `config/several-configs` issue (error, D175) with `path` = the directory, naming the files (D202).
- **Loading:** `.tsshowcase` and `.tsshowcase.json`/`.jsonc` with `jsonc-parser` (§4; an empty `.tsshowcase` is `{}`); `.tsshowcase.js`/`.mjs`/`.ts`/`.mts` through jiti with the same aliases (§8.3), taking the default export: a plain object or `true` (a promise of one allowed; typically wrapped in `defineDirectoryConfig`); a function is rejected (as for configs, D115).
- **Validation:** ajv against `directory-config.schema.json` (§5); a schema violation (incl. `false`, a bad `slug`) is a thrown `ConfigError` naming the file (D175); then the form checks of §5 "Map keys and patterns" (CC9). Kind values are checked against the well-known and the module's kinds (D189).
- **Reloading (QH):** callable again after every `dev` rescan; JS/TS forms load with jiti's module cache off (a fresh evaluation per call, imports included), so an edited `.tsshowcase.ts` never returns stale content.
- **Result:** per directory, its file's value (`true` normalised to `{}`), with the directory as its base; **no merging** and no other `config/*` issues here: C9 merges every source per directory (D188: root `directories` entry < package top level < package `directories` entry < file), raises `config/shadowed`, `config/missing-path`, … and places them (D180) (CC1).

## 5. The config schema (D58, D62, D66, C2 §10)

Three plain JSON Schema files with committed tsschema sidecars, in the `config-schema/` area. `additionalProperties: false` everywhere. Documentation in each property's `description` (shown by editors via `$schema`). Defaults are stated in the descriptions, not as `default` keywords (§4).

**Root config (`config.schema.json`, redesign keys marked):**

| Key                 | Type                                                                                           | Default (applied in code)               | Notes                                                                  |
| ------------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------- | ---------------------------------------------------------------------- |
| `$schema`           | string                                                                                         | –                                       | ignored                                                                |
| `module`            | string                                                                                         | `'react-default'`                       | reference (D9, §8)                                                     |
| `moduleOptions`     | object                                                                                         | `{}`                                    | validated by the module's `optionsSchema` (§9)                         |
| `themes`            | string \| string[] (min 1)                                                                     | `['default-light', 'default-dark']`     | references, optional `#name` (§10)                                     |
| `defaultTheme`      | string                                                                                         | –                                       | a theme name; checked by C11                                           |
| `outDir`            | string                                                                                         | `./showcase` (relative to the cwd)      | relative to the config base; `--out` wins (D167); an explicit value is a hard ignore (D101) |
| `scope`             | string[]                                                                                       | `['.']`                                 | **package-relative** paths and globs, the default for every package (D165, C4 §5) |
| `ignore`            | string[]                                                                                       | `[]`                                    | gitignore syntax incl. `!`, root-relative (D103, D107, C4 §4)          |
| `rules`             | object of `'error' \| 'warning' \| 'off'`                                                      | `{}`                                    | keys are rule ids **or patterns** (D176)                               |
| `testRunner`        | string \| string[]                                                                             | –                                       | **redesign:** pins the runner(s), skips detection (D172); root only    |
| `testRunners`       | string[]                                                                                       | `[]`                                    | extra candidates (D172, §11); root only                                |
| `strict`            | boolean \| (`'check'` \| `'tests'` \| `'unavailable'` \| `'unhandled'` \| `'warnings'`)[]          | `false`                                 | **redesign:** D164; `true` = the four default parts, `false` and `[]` none; root only |
| `tests`             | boolean                                                                                        | `true`                                  | `false` = `--no-tests` (D197)                                          |
| `directories`       | object: repo-relative directory → directory config (object) or `true`                          | `{}`                                    | **redesign:** D188; the directory-config schema per entry              |
| `kinds`             | object: literal `'<path>#<name>'` → kind                                                       | `{}`                                    | **redesign:** D189; paths repo-relative                                |
| `exempt`            | object: `'<path or glob>[#name]'` → (`'docs'` \| `'tests'` \| `'showcase'` \| `'*'`)[]        | `{}`                                    | **redesign:** D191                                                     |
| `vcs`               | object: `webUrl`, `directory`, `host` (`github` \| `gitlab` \| `bitbucket`), `fileUrlTemplate`, `ref` | –                                 | D110, D205, C4 §9                                                      |
| `dev`               | object: `host` (string), `port` (integer 0–65535)                                              | module/toolchain default                | K8 (Q5)                                                                |

**Removed by the redesign:** `check` (D164), `failOnTestFailure` (D164), `units` (D188), `moduleOptions.testRunner` (D172; the module's own schema no longer has it). They were never published, so they are ordinary unknown keys, without migration hints (CC7). **`collections` at the top level** of a root or package config is a schema error with a dedicated message: "`collections` can only be set in a directory config or a `directories` entry" (CC7).

**Package config (`package-config.schema.json`):** `$schema`, `skip` (boolean), `scope` (package-relative, replacing the root default for this package; D165), `rules`, `moduleOptions` (D59, D69, D148), `directories` (keys relative to the package), `kinds`, `exempt` (D188, D189, D191). Any root-only key there (`themes`, `testRunner`, `testRunners`, `strict`, `tests`, `vcs`, `dev`, `outDir`, `ignore`, `module`, `defaultTheme`) gets a dedicated error: "`themes` can only be set in the root config (`<path>`)".

**Directory config (`directory-config.schema.json`, redesign, D188):** an object or `true` (`false` is a schema error): `paths` (string[]; no leading `!`), `name` (string), `slug` (URL-safe: `^[a-z0-9]+(?:-[a-z0-9]+)*$`, CC10), `order` (number), `kinds` (literal `'<path>#<name>'` keys), `collections` (object: key → string[]), `exempt`. Keys relative to the config's directory. Published next to the other two (C2 §10), with an editor `json.schemas` tip in the CLI README (C13). **Shared definition (QJ):** written once in `directory-config.schema.json`; the `schemas` codegen step (C1 A1) copies it into a `$defs.directoryConfig` of `config.schema.json` and `package-config.schema.json`, whose `directories` values reference it locally (`$ref: '#/$defs/directoryConfig'`), so every published file is self-contained for editors, ajv and tsschema; `ci:schemas --check` catches a stale copy.

**Map keys and patterns (CC9),** checked in code after the schema, each failure a `ConfigError` naming the file and key: `directories` keys are literal relative directory paths, normalised like `scope` entries (leading `./` and trailing `/` removed, `.` = the config's own directory) and rejected when absolute, containing `..` or a backslash (glob characters are literal; glob keys are deferred, Z10); `kinds` keys are `<path>#<name>` with a non-empty literal path and name (split at the last `#`); `collections` keys (QG) are non-empty, not `.` or `..`, and contain no `/`, `\`, `#` or glob character (`* ? [ ] { } ( ) !`), anything else a file base could be is allowed (`Button`, `use-thing`, `Grid.Row`); `exempt` keys and `paths`/collection members may be globs, but a leading `!` is rejected (no negations in v1, D188), and `..` segments are allowed only as a leading run (`../x`, `../../src/*`; `a/../b`, `**/../x` rejected, QF). Whether a key names an existing, in-scope directory, file or symbol is C9's (`config/unused-key`, `config/missing-path`, `config/out-of-scope`).

**Directory-config semantics (the contract C9 implements; QF, D188, D190, D191).** Written once here, mirrored in the schema `description`s and the CLI README (§15); C9 cites it.

- **Where:** a `.tsshowcase` (JSONC; empty = `{}`) or `.tsshowcase.{json,jsonc,js,mjs,ts,mts}` file in an in-scope directory, or a `directories` entry in the root config (keys relative to the repo root) or a package config (keys relative to the package). An entry for a directory that isn't in scope: `config/out-of-scope`. Several files in one directory: `config/several-configs`, none applies (§4.1).
- **Precedence for one directory:** the root config's `directories` entry < the package config's top level (package directory only: `kinds`, `exempt`) < the package config's `directories` entry < the directory's own file. Each key of a later source **replaces that key** of the earlier ones wholesale (no deep merge), with a `config/shadowed` warning per key set in more than one source.
- **Nodes:** a directory config creates an (indirect) node unless all its keys are non-node keys (`kinds`, `collections`, `exempt`); empty, `{}`, `true`, display-only and `paths` configs all create one; no `paths` = `paths: []` (a grouping node).
- **Patterns** (`paths`, collection members, `exempt` keys): picomatch relative to the config's directory, `dot: true`, globs match files only; no negations in v1; `#name` split off at the last `#` before matching (`#name` literal); `../` only as a leading literal run, staying within scope and the package; directory names containing glob characters must be escaped in globs; patterns the CLI builds itself are escaped.
- **`paths` entries** list symbols and resources, all primary (D173): a literal exporting file lists all its symbols; `file#name` one symbol (an unknown name: `config/missing-path`); a collection key lists the collection, a collection member only that member's own symbols; a glob lists every match (none: `config/missing-path`); docs, tests and showcases are listed as resources; a literal directory: `config/no-role` with the hint "use `sub/**`"; a literal entry with no role: `config/no-role`; a missing path: `config/missing-path`; anything out of scope or in another package: `config/out-of-scope`, ignored; upward entries within scope allowed (they count for the check, not for link targets, D179). Listing an exporting file doesn't bring its name-matched resources along.
- **`kinds`** (D189), **`collections`** (D190), **`exempt`** (D191): their own decisions; `collections` only in directory configs and `directories` entries.
- **Display keys:** `name`, `slug` (the pattern above; duplicate sibling slugs: `config/slug`), `order`.

**Inferred types:** `TsshowcaseConfig`, `PackageConfig` and `DirectoryConfig` from the sidecars (C2 §10), never hand-written; the map-shaped keys (`rules`, `moduleOptions`, `directories`, `kinds`, `exempt`, `collections`) infer as `{ [key: string]: unknown }` until tsschema infers records (D58, the known gap before `1.0.0`). `defineConfig`, `definePackageConfig` (P6) and `defineDirectoryConfig(config: DirectoryConfig | true)` are identities.

**Publishing (C2 §10, D62):** the schema files and their sidecars are emitted into `dist` (C1 A1). The README documents `"$schema": "./node_modules/@ofzza/tsshowcase/config.schema.json"`, which needs a real file at the package root: the CLI package's own `build` script copies the three schema files to the package root after `tsc` (gitignored, listed in `files` next to `dist`). The `exports` subpaths `./config.schema.json`, `./package-config.schema.json` and `./directory-config.schema.json` have a `source` condition pointing at `src/config-schema/*.json` (so in-repo imports work without a build, C1 §3) and a `default` pointing at the root copies (P11). This deviates from C2 §10 ("no emit step", a `config/` directory) and C1's `files: ["dist"]`; both get wording-only notes (R6).

## 6. Precedence and path bases (M12, Q3)

**Precedence:** built-in defaults < root config < package config (package keys only) < flags (D6, D79).

| Flag | Config key | Arrays | Paths relative to |
| ---- | ---------- | ------ | ----------------- |
| `--cwd <dir>` | – | – | process cwd |
| `--config <file>` | – | – | process cwd |
| `--module <ref>` | `module` | – | `file:` references (§8.1): flag: process cwd; config: config base |
| `--theme <ref>` (repeatable) | `themes` | **replaces** | as `--module` |
| `--default-theme <name>` | `defaultTheme` | – | – |
| `--scope <glob>` (repeatable) | `scope` (root and package configs) | **replaces** the root and every package config's value (C4, `04b` Q2) | **each package** (D165) |
| `--ignore <pattern>` (repeatable) | `ignore` | **appends** | repo root (both) |
| `--vcs-web-url`, `--vcs-directory`, `--vcs-host`, `--vcs-file-url-template`, `--vcs-ref` | `vcs.*` | – | – |
| `--strict`, `--strict=<part>` (repeatable), `--no-strict` (check, build; QA) | `strict` | **replaces** (§3) | – |
| `--rule <id>=<severity>` (repeatable; check, build, dev; QA) | – (consulted before `rules`, D176) | – | – |
| `--no-tests` (build, dev; C6) | `tests` | – | – |
| `--out <dir>` (build; C6) | `outDir` | – | flag: process cwd; config: config base (D167) |
| `--host`, `--port` (dev; C6) | `dev.host`, `dev.port` | – | – |
| – | `directories`, `kinds`, `exempt` | – | keys: repo root (root config), the package (package config), the directory (directory config); an exception to D112, like `scope`/`ignore` (D188) |
| module flags (`optionFlags`, D79) | `moduleOptions.*` | per schema | path-marked options: flag: process cwd; root config: config base; package config: its directory (§9.4, D116) |

**Action-restricted flags (QA):** a flag is accepted only where it has an effect; elsewhere it is an "unknown flag for `<action>`" error. `--strict`/`--no-strict`: `check` (only `warnings` matters there) and `build` (`dev` never fails on issues: a long-running server can neither refuse to start nor stop mid-edit, D164; `init` runs no check). `--rule`: the actions that print issues (`check`, `build`, `dev`). `--no-tests`: `build` and `dev`, the actions that run tests (`check` and `init` only list them, D197). The config keys are shared by every action and never an error: `check` and `init` ignore `tests`, `dev` and `init` ignore `strict`.

Rationale for the arrays (M12): `--ignore` appends because the common use is "this run, also leave out X"; `--scope` replaces because the common use is "only look at X". `--ignore` stays root-relative like its key; `--scope` is package-relative like its key since the redesign (D165), so a pattern means the same wherever it is written.

`--theme` replaces, so `--theme default-dark` gives a single-theme build.

## 7. Per-package configs (D59, D69)

- **Discovery:** for `layout: 'workspace'`, the same six file names in each `WorkspaceInfo.packages` directory that **exists** (C4's `packageExists()`, `04b` Q3: a package ignored by an ignore file, config `ignore`, the output marker, `outDir` or a skipped ancestor is never configured, so a broken config inside it never stops a run); several in one package is an error. Loaded like the root config (jiti, aliases), validated against the package schema. Config files are metadata: read whether or not anything of the package is in `scope` (D101).
- **`skip: true`** puts the package into `skippedPackages` (C4: a hard cut, nested packages included).
- **Merging `moduleOptions`:** path-marked option values are first made absolute against each source's own base (§9.4, D116); then the package's marked keys are **shallow-merged** over the root values (C2 §10); module flags are then applied on top of every merged result, then each result is validated against `optionsSchema` (D69, D79). A validation error names the package config file it came from.
- **Not merged (redesign, CC4):** `rules` stay per source (flags > package config > root config, the first source with a match decides, D176); `scope` replaces the root default (D165); `directories`, `kinds`, `exempt` are handed to C9 per source with their base, because they merge per directory with the directory configs (D188).
- **Per-package module options (D69, D136, D145, D148):** module options are root-only by default. In a package config's `moduleOptions`, only top-level properties the module's `optionsSchema` marks with `"x-tsshowcase-per-package": true` (`PER_PACKAGE_OPTION_KEYWORD`, C2 §5.11) are merged; any other key is warned about ("`<key>` can only be set in the root config") and dropped before merging and validation. The keyword is registered with ajv as an annotation. No option names are hard-coded in the CLI; `viteConfig`, `showcaseStylesheets` and `showcaseSetup` are root-only because `-base-react` doesn't mark them (`testRunner` is no longer a module option, D172).
- **Single-package layout:** no package configs (the root config is the package's config).

## 8. Plugin references and loading (D9)

### 8.1 Reference syntax and resolution order (Q2)

**A path must carry a `file:` prefix (Q2):** the prefix alone decides that a reference is a path, so it is always explicit. D9's order applies to the rest:

1. **Path:** `file:<path>`, like npm's `file:` dependency specifiers: everything after `file:` is a filesystem path, relative (`file:./tools/my-module.ts`, `file:../shared/theme`, `file:tools/x.ts`) to the flag's or config's base (§6), or absolute (`file:/opt/plugins/x.js`, `file:C:/plugins/x.js`). The standard URL forms `file:///…` and `file://localhost/…` are accepted as absolute paths. A directory is resolved like a package (its `package.json` `exports`, else `index.*`). Theme references split off `#<name>` at the last `#` (§10).
2. **Built-in name** (no slash, not scoped): the table in `plugins/builtins/`, per kind. v1: module `react-default` → `@ofzza/tsshowcase-module-react-default`; themes `default-light`, `default-dark` → `@ofzza/tsshowcase-theme-default#default-light|dark`; runner `vitest` → `@ofzza/tsshowcase-testrunner-vitest`. Built-ins always resolve from the **CLI's own dependencies**, so they match the CLI's version.
3. **npm package** (bare or scoped specifier, optional subpath): resolved with ESM semantics (`exports` conditions `import`, `default`; no `source`) **from the repo root first, then from the CLI's own location** (so a globally installed CLI with globally installed plugins also works). Resolution uses jiti's ESM resolver with a parent URL (an `exports`-only ESM package is part of the tests). npm-resolved plugins load as published JS (§8.3), so a TS-only workspace package holding a custom plugin is referenced with `file:` or built first (documented, P9).
4. Anything else fails with every attempt listed. A reference that looks like a path without the prefix (`./x.ts`, `../x`, `/abs/x.js`, `C:\\x`, or a bare name that exists as a file or directory under the base) gets the hint "to load a path, write `file:./x.ts`"; it is never loaded as a path.

Refines D9: D9's order "built-in → npm → path" stays, but a path is recognised only by the `file:` prefix instead of being tried last, so a typo in a package name never silently picks up a same-named file or directory, and a reader of the config sees at a glance which plugins are local code.

### 8.2 Kinds and shape checks

Each reference is loaded for a known **kind** (module, theme set, runner). The default export must be a class (a function with a `prototype`); its static `apiVersion` must be a valid semver range (`semver.validRange`), and `semver.satisfies(IMPLEMENTED_API_VERSION, apiVersion, { includePrerelease: true })` must hold (D13, D80; prereleases such as `0.2.0-next.0` satisfy `^0.2`, P7). After construction the instance is shape-checked for its kind (module: `analyze` is a function and every optional method that is present is a function, D198; theme set: `themes` array; runner: `detect`, `listTestFiles` (required since the redesign, D172), `run`, `watch`). Errors name the reference, the resolved file and what to do, e.g. "module `file:./tools/my-module.ts` targets tsshowcase API `^0.2`, but this CLI implements `0.1.3`: upgrade `@ofzza/tsshowcase` or use a module version written for 0.1".

### 8.3 jiti and the CLI's packages for path-loaded plugins (L12, D117)

- **Path references** (TS or JS) load through **jiti** (one instance per invocation, ESM interop, `interopDefault`), so custom plugin authors need no build step (D9). jiti **aliases every `@ofzza/tsshowcase*` package in the CLI's dependency tree** (`-api`, `-base-react`, `-base-vite`, `-theme-default`, `-testrunner-vitest`, `-module-react-default`, the CLI itself) **to the CLI's own copies** (P4, D117). A path-loaded plugin therefore works under `npx`, pnpm and when only the CLI is installed; a custom layout module extending `-base-react`'s base class (D48) gets the base and `-api` from the same tree, never two contract copies. Other npm dependencies of a path-loaded plugin resolve normally from its own location. Documented for authors (C13).
- **npm packages and built-ins** load with native `import()` of the resolved file: they are published JS and keep their own dependencies (whose ranges npm resolved); no aliasing.
- jiti's transpile cache goes to the CLI cache directory (`<root>/node_modules/.cache/tsshowcase/jiti`, C11 owns the directory) **only when `<root>/node_modules` exists**; otherwise jiti's own default in the OS temp directory, so the CLI never creates `node_modules/` in a repo that has none (P10). `dev` doesn't reload plugins, the root config or package configs: a change to any of them restarts (C4's recorded C11 point), and so does a change to the set of existing packages after a rescan (QH: C11 compares it with the set C6 resolved). **Directory configs** are the exception: `loadDirectoryConfigs` may be called again after any rescan, and their JS/TS forms load with jiti's module cache off, so each call evaluates the file and its imports afresh (QH).

### 8.4 `PluginHost`

Every constructed plugin receives `{ apiVersion: IMPLEMENTED_API_VERSION, cliVersion, logger }`, the logger prefixed with the plugin's display name (`displayName`, else the reference).

## 9. The module

### 9.1 Loading

`LoadedModule = { reference, url, constructor: ShowcaseModuleConstructor, instance: ShowcaseModule }`. The static members are validated at load: `optionsSchema` must compile with ajv (a compile error is a module error); `rules` ids must be unique and **outside the CLI-reserved namespaces** `represent/`, `tests/`, `config/`, `docs/`, `scan/`, `module/` (02b H4, C2 §9); every `optionFlags` target must be a supported schema path (§9.2); **`kinds`** (redesign, D162): no key may collide with a well-known kind (`component`, `hook`, `function`, `class`, `constant`, `type`), every key matches `^[a-z][a-z0-9-]*$` (safe as a `data-kind` value, 03c, CC5), `requires.docs`, `requires.tests`, `requires.showcase` are `required` or `optional` when present, other `requires` keys ignored (C2 §5.1: new resource types arrive later); **`testRunners`**: an array of strings (resolved in §11). Each failure is a load-time `PluginError` naming the module.

### 9.2 Module options and `optionFlags` (D79)

- **ajv** (`ajv/dist/2020`), `allErrors: true`, `useDefaults: true` (C2 §5.11: defaulted properties are required in `OptionsOf`), `strictSchema: false` for module schemas (third-party schemas may carry editor keywords), `strict: true` for our own. Options are deep-cloned before validation (`useDefaults` mutates). No schema: options pass through as given.
- **Flags from the schema:** the target path is followed through `properties` (and local `$ref`s); the property's type decides the flag: `boolean` → `--x` / `--no-x`; `number`/`integer` → coerced; `string` (incl. `enum`, validated by ajv) → value; an array of those → repeatable. Objects, unions and `oneOf`/`anyOf` targets are a load-time error naming the module and the flag. The schema `description` becomes the help text. Flag names must be kebab-case.
- Flags apply to the root options and to every merged per-package result (§7), then validation runs.

### 9.3 Instantiation

After options are validated: `new Module({ ...pluginHost, options, repoRoot })`. A throwing constructor is a module error naming the module.

### 9.4 Path-valued options (D116, P1, C2 amendment B5)

Module options may hold paths (`showcaseStylesheets`, `showcaseSetup`, `viteConfig`, C3 §1, D138) and plugin references (a module's own option naming a plugin; `testRunner` is no longer one, D172). After root config, package configs and flags are merged, nobody could tell which base a relative value was written against, so **modules mark them in their schema** and the CLI resolves them **per source, before merging**:

- **`format: 'tsshowcase-path'`** (constant `OPTION_FORMAT.path` in `-api`, B5) on a string property: the value is a filesystem path; relative values are made absolute against the source's base (root config: the config base; package config: its directory; flags: the process cwd, D112).
- **`format: 'tsshowcase-reference'`** (`OPTION_FORMAT.reference`): the value is a plugin reference (§8.1); only `file:` values are rewritten (`file:./x.ts` → `file:/abs/x.ts`), built-in names and npm packages pass unchanged.
- The marker may sit on a property, on array `items` or on `additionalProperties`, reached through `properties`/`items`/`additionalProperties` and local `$ref`s. A marker under `oneOf`/`anyOf`/`allOf`/conditionals is a load-time error naming the module (the CLI couldn't tell which branch applies). ajv registers both formats (any string is valid; the marker only drives the rewrite).
- **Modules therefore only ever see absolute paths** (and absolute `file:` references).
- Absolute paths in options never reach the site by themselves; a module that embeds an option value in its output keeps C2 rule 2 (no leaks) itself.
- Documented in the `-api` `module/` README (how to mark options) and the CLI README (where relative paths in module options are relative to). Our own modules mark theirs (C8, C10).

## 10. Themes (K14: C6's half)

- **Parsing:** each `themes` entry is `<ref>` or `<ref>#<name>` (C2 §7); the built-in names are aliases for `@ofzza/tsshowcase-theme-default#<name>`. Parsed in `resolveConfig`, so syntax errors fail early; `ThemeReference = { reference, base, name? }`.
- **`loadThemes()`** (build and dev): loads every referenced theme set once (two references to the same set share an instance), version-checks it, validates each `ThemeDefinition`: `name` matches `^[A-Za-z0-9][A-Za-z0-9_-]*$` (it lands in an attribute selector and in `localStorage`), `scheme` is `light` or `dark`, `css` is a non-empty array of absolute paths to existing `.css` files. `<ref>#<name>` must name a theme in the set (the error lists the available names). Returns `ResolvedTheme[]` in config order, the CLI's own type since plan 7 (no `-api` contract uses it, D102): `interface ResolvedTheme extends ThemeDefinition { readonly reference: string }` (the config reference it came from).
- **Not here (C11):** duplicate names across references, `defaultTheme` existing, scoping, the cache.

## 11. Test runners (D172, replacing `loadTestRunner`, D42, D74)

The CLI owns the runners (D172); modules never call them. `resolved.loadTestRunners(repo)` (called once after the scan by every action: `check` and `init` list, `build` and `dev` list and run; CC3) returns:

```ts
export interface LoadedRunners {
  readonly runners: readonly LoadedRunner[]; // in candidate order
  readonly issues: readonly Issue[]; // tests/runner-failed for candidates that failed to load or whose detect threw (CC2)
}

export interface LoadedRunner {
  readonly id: string; // display name, de-duplicated in candidate order: 'vitest', 'vitest-2' (D172)
  readonly name: string; // displayName, else (QE, never a path): the built-in name; the npm package name plus subpath; a `file:` file's base name without extensions; a `file:` directory's package.json `name`, else its directory name
  readonly reference?: string; // as written, unless a filesystem path (Representation.tests, D186)
  readonly runner: TestRunner;
}
```

- **Candidates, in order:** the module's static `testRunners` (each resolved with the **module's file as the parent**, C2 §5.1, CC6: built-in names as built-ins, from the CLI; npm names from the module's own location only, since the module depends on them; relative `file:` paths against the module file's directory), then the built-ins (`vitest`), then config `testRunners` (config base for `file:`). **De-duplicated by identity, keeping the first position (P8):** built-in names map to their package, npm references compare by package name plus export subpath, `file:` references by absolute path.
- **Pinning:** config `testRunner` (one reference or a list) uses exactly those runners and skips `detect`. **Resolution (QD):** a pinned npm name equal to one of the module's static `testRunners` (same package name and subpath) resolves exactly like that static, from the module's location; any other pinned reference resolves like a config reference (built-in names as built-ins, npm from the repo root then the CLI, `file:` against the config base). **Only the pinned runners are loaded:** the unpinned candidates are never loaded or detected, so they raise no issues; a pinned runner that fails to resolve, load, version-check or shape-check is a thrown `PluginError` (the user asked for it explicitly; CC2).
- **Detection:** otherwise every candidate whose `detect(repo)` resolves `true` is used (several are fine since the redesign). A candidate that fails to load, or whose `detect` throws, gives a `tests/runner-failed` issue (error, D175) naming it, and the others are still used (CC2).
- **No runner** and the representation's consequences (`tests/no-runner`, the "nothing requires tests" short-circuit) are C9's (D172).
- Instances are cached per identity for the invocation; `detect` runs once per invocation (in `dev`, the runner set is fixed for the session, C11).

## 12. Scanner inputs, VCS and rules

- **`scope`:** per package, the effective entries (`--scope` > the package config's `scope` > the root config's `scope` > `['.']`; C4's recorded point), validated with C4's `validateScope()` per source (package-relative; absolute paths, `..` and backslashes rejected, `04b` Q2); a `ScopeEntryError` becomes a `ConfigError` naming the source (the config file or `flag --scope`); the entries are handed to `ScanOptions.scope` **as written**, and `scanRepo` normalises them, so its unmatched warning quotes them as written (QI). Unmatched entries are C4's printed warning (C4 §5), no longer an issue.
- **`ignore`:** passed through (gitignore syntax is validated by C4's matcher; an unparseable pattern names its source: the config file or `--ignore`).
- **`skippedPackages`:** from package configs (§7), parents first.
- **`outDir` (M13, P2, D167):** `--out` (relative to the cwd), else config `outDir` (relative to the config base), else `./showcase` relative to the cwd. Only an **explicit** value is passed to the scanner (D101). Validated early with the same check `scanRepo` uses (C4 §10: not the root, not an ancestor of it, not containing a package), shared as one helper exported internally from `scan/`.
- **`vcs`:** the five keys and flags merged (flag → config) into C4's `VcsOverrides`; `vcs.host` must be one of the three hosts; `fileUrlTemplate` must contain `{path}` (and may contain `{ref}`, D205); `vcs.ref` must be a non-empty string without whitespace; `HEAD` and `CURRENT` are recognised case-insensitively and normalised to upper case (D205, CC8).
- **Rule ids (D175, D176):** keys of `rules` (root and package configs) and `--rule` ids may be exact ids or patterns. **Pattern syntax (QC):** prefix patterns only: a key either has no `*` (an exact id) or ends with one `*` matching any characters, `/` included (`represent/*`, `represent/missing-*`, `*`); a `*` anywhere else is a `ConfigError`; the literal prefix (the text before the `*`) is what D176's "longer literal prefix wins" compares. A key whose literal text pins it to a CLI-reserved namespace is checked against the CLI's table; any other key (`react/*`, `rep*`, `*`) is matched against the CLI's table plus the module's declared rules and warned about only when it matches nothing **and** the module declares `rules` (P14). A key in a **CLI-reserved namespace** (`represent/`, `tests/`, `config/`, `docs/`, `scan/`, `module/`) is checked against the CLI's table: `represent/missing-docs|tests|showcase`, `tests/no-runner`, `tests/runner-failed`, `config/several-configs|collection-clash|collection-membership|kind-conflict|same-base|unused-key|missing-path|no-role|out-of-scope|shadowed|slug`, `docs/broken-link|broken-anchor`, `module/error`, `scan/invalid-manifest|no-manifest|unreadable` (no `scan/scope-unmatched` any more); a key in any other namespace is checked against the module's `rules` when it declares them and accepted silently when it doesn't (P14, 02b Q4). A key matching nothing prints a **warning** naming its source (not an issue, G5). Severities are applied by C9. The CLI's table lives **once**, as an internal constant (id → default severity) used by this validation and by C9 (D56, one place per fact); the list above only mirrors C2 §9.

## 13. Errors

- **`ConfigError`** (thrown; C9 prints it and exits non-zero): the file (or `flag --x`), the JSON path (`moduleOptions.viteConfig`), a message and an optional hint. ajv errors are reformatted (one line per error, deduplicated, `additionalProperties` errors name the property).
- **"Did you mean":** an unknown key gets the closest known key (edit distance ≤ 2). Fixed hints exist only for **mistakes a new user could plausibly make** (CC7): `exclude` in a package config → `skip` (D69); a `build` positional → "did you mean `--out <dir>`?" and `build --strict tests` → "did you mean `--strict=tests`?" (D167, §3); `collections` at the top level of a root or package config → "can only be set in a directory config or a `directories` entry" (§5). Names that were never published (`include`, `exclude` at the root, `vcs.defaultBranch`, `check`, `failOnTestFailure`, `units`, `--no-check`, `--fail-on-test-failure`) get no migration hints: nobody has them in a config, and they would leave dead names in the code and tests (`0.x`, D80).
- **`PluginError`:** a plugin that fails to resolve, load, version-check, validate or construct; names the kind, the reference and the resolved file. Shared with C9/C11 for later plugin failures (`module/error`).

## 14. Fixtures and tests

**Stub plugins (L14):** C6's tests never depend on the built-in packages' contents (C3 and C7 are implemented later): built-in name lookups are tested with the table injected. The real built-in names get their tests with C3 and C7.

| Fixture                                                  | Purpose                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `plugins-stub`                                           | Stub plugins in TS and JS importing `@ofzza/tsshowcase-api` (the alias): a module with `optionsSchema`, `rules` and `optionFlags`; a module without a schema; a theme set with two themes and CSS files; runners whose `detect` returns true / false / throws; a module with path- and reference-marked options (incl. arrays and a `$ref`) and one with a marker under `oneOf`; a module with `kinds` and `testRunners` statics (and failing variants); runners with `listTestFiles` (and one without); a layout module extending a stub base package aliased like `-base-react`; failure stubs (non-class default export, invalid range, unsatisfied range, missing `analyze`, a colliding flag (incl. one colliding only with another action's flag, and a `no-` name), an unsupported flag target, a throwing constructor) |
| `config-ts`, `config-json`, `config-jsonc`, `config-mjs` | One config per format, each using `defineConfig` or `$schema`, pointing at the stub plugins                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `config-several`                                         | Two config files (error)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `config-workspace`                                       | A workspace with package configs: `skip`, `rules`, `scope`, `directories`, `kinds`, `exempt`, `moduleOptions` (incl. relative path-marked values), one with a root-only key (error), a `tsshowcase.config.ts` using `definePackageConfig`, a package ignored by `.tsshowcaseignore` whose config is broken (never loaded); directory configs in every form, and a directory with two                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `config-package-at-root`                                 | A config with `skip` at the root: the L4 error standalone and, run on a package of a workspace, with the workspace-root hint (P12)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

npm-installed stubs (a plugin in `node_modules`, an `exports`-only ESM package, one installed next to the CLI only, a copy of a built-in runner package installed in the repo, P8) are created at test time with `copyFixture`'s `files` option.

Tests, per directory of §1:

- argv: both phases, unknown flags and extra positionals in each phase, positional arity, repeatable and negatable flags, number coercion, collisions against every action's flags, `help`/`version`, `no-` names and the `vcs-` prefix (the same result under every action), `describeFlags` with a loadable module, with invalid `moduleOptions` (still lists the module's flags) and with an unloadable module (CLI flags plus the warning);
- config files: each format, JSON and JSONC with `//` and `/* */` comments (incl. comment-like text inside strings kept), trailing commas rejected, JSON syntax errors with line and column, `--config`, several files, none, default export checks, the aliases (a config importing `defineConfig` from a copy without a local install), validation errors and their formatting, every "did you mean" hint, the L4 error;
- precedence and bases: every row of §6, `--ignore` appending and `--scope` replacing, `outDir` from `--out` > config > default and explicit vs default, a `build` positional rejected with the `--out` hint, `outDir` early validation; path- and reference-marked options from the root config, a `--config` elsewhere, a package config and a flag each resolved against their own base before merging, unchanged built-in and npm references, a marker under `oneOf` rejected;
- package configs: discovery, several in one package, `skip`, shallow merge, flags on top, per-package validation errors naming the file, root-only keys, module options not marked `x-tsshowcase-per-package` warned about and dropped (D148);
- references: every `file:` form (relative with and without `./`, absolute, `file:///`, a Windows drive, a directory, a theme `#name` after a path), built-in table, resolution from the repo then the CLI, `exports`-only ESM packages, the missing-prefix hint for every path-like form (never loaded), Windows-style paths normalised;
- loading: version checks (valid/invalid/unsatisfied ranges, `^0.1` vs `0.2.0`, a prerelease `0.2.0-next.0` against `^0.2`), shape checks, constructor errors, `PluginHost` contents, the aliases for path-loaded plugins (`-api` and a base package resolve to the CLI's copies) and **no** alias for npm-loaded ones; the jiti cache in the repo only when `node_modules` exists;
- module options: defaults applied, validation errors, no schema, `optionFlags` for every supported type, unsupported targets, flags on root and package options;
- themes: parsing, `#name` selection and its error, shared instances, definition validation, version checks;
- test runners (§11): candidate order (module statics, built-ins, config), resolution of module statics against the module file, a pinned reference matching a module static resolved from the module's location, pinning leaving a broken unpinned candidate unloaded (no issue), de-duplication by identity (a built-in runner also installed in the repo counted once), pinning one and several, a pinned runner failing to load (thrown), several detecting runners all used, ids de-duplicated (`vitest`, `vitest-2`), fallback names without `displayName` for each reference type (no path in `name`/`id`), a candidate failing to load and a throwing `detect` (`tests/runner-failed`, the others used), `listTestFiles` required by the shape check, caching;
- argv normaliser: bare `--strict` (rewritten to four tokens; a following word stays positional), `--strict=<part>` repeated, the union with a bare `--strict`, `--no-strict`, order (`--strict --no-strict` → none, `--no-strict --strict=warnings` → `warnings`), tokens after `--` untouched, a repeated `--rule` id (later wins), empty and unknown parts, config `strict` replaced by any flag, `--rule` split on the last `=` (incl. `--rule=-x=off`), patterns, the positional hints (`build --strict tests`, `build out`);
- rule ids: module ids, the CLI's table, prefix patterns (`*` only at the end; elsewhere rejected; longer literal prefix wins), keys such as `rep*` and `*` matched across namespaces, reserved namespaces always checked, others only with module `rules`, unknown keys printing a warning; module rules in a reserved namespace rejected at load;
- module statics: `kinds` colliding with a well-known kind, a key failing the pattern, invalid `requires` values; `testRunners` not an array of strings;
- `scope` validation per source (`ScopeEntryError` wrapped with the source named), entries handed over as written;
- package configs: only existing packages configured (`packageExists()`), a broken config inside an ignored package never loaded; `scope` override and `--scope` replacing both;
- directory configs (§4.1): every file form (`.tsshowcase` JSONC, empty, `.json`, `.jsonc`, `.js`, `.mjs`, `.ts`, `.mts` via `defineDirectoryConfig`), `true`, `false` rejected, a function rejected, schema errors thrown with the file, several files → `config/several-configs` with the directory as `path`, unknown kind values; no merging; a second call after editing a `.tsshowcase.ts` (and a file it imports) returns the new content;
- `vcs.ref` (CC8): empty and whitespace rejected, `head`/`current` normalised to upper case, a branch with `/` kept, `{ref}` accepted in `fileUrlTemplate`;
- map keys (CC9, QF): `..` in `paths`, collection members and `exempt` keys only as a leading run (`../../x` accepted; `a/../b`, `**/../x` rejected); `directories` keys absolute, with `..` or a backslash rejected, `./`, a trailing `/` and `.` normalised; `kinds` keys without `#` or with an empty part rejected, split at the last `#`; `collections` keys empty, `.`, `..` or containing `/`, `\`, `#` or a glob character rejected, `Grid.Row` accepted; a leading `!` in `exempt` keys, `paths` and collection members rejected; `slug` accepted and rejected cases (CC10); kind values inside `directories` entries;
- root-config redesign keys (`directories`, `kinds`, `exempt`, `strict`, `testRunner`, `testRunners`) in the `config-*` fixtures; root-only keys (`strict`, `testRunner`, `testRunners`, `tests`) rejected in a package config; `strict: []` selecting none;
- step order: an invalid explicit `outDir` rejected before any package config loads; `scope` and options entries exactly for existing, non-skipped packages;
- hints (CC7): `exclude` in a package config, the two `build` positional hints, `collections` at the top level; a removed key (e.g. `failOnTestFailure`) is an ordinary unknown key;
- the shared directory-config definition (QJ): the `$defs.directoryConfig` copies match `directory-config.schema.json` (`ci:schemas --check`); tsschema infers `directories` values through the local `$ref` (type test);
- the exported areas: `defineConfig`, `definePackageConfig` and `defineDirectoryConfig` (type tests: a config object type-checks, a wrong key fails, `skip` only in a package config, `collections` only in a directory config, `true` accepted), `configSchema` validates a sample config and rejects an invalid one (C2 acceptance), a consumer-style type test over `dist` infers `TsshowcaseConfig` as an object type, not `never` (C2 acceptance, D62), the package-root schema copies match the sources, the `source` export condition resolves the schemas without a build.

## 15. Documentation

- CLI README: the config file (formats, discovery, `--config`, `$schema`, `defineConfig`), every key with default and flag (§5, §6), precedence and path bases, the `scope` (package-relative globs) vs `ignore` (root-relative gitignore syntax) asymmetry (N16, D165), per-package configs, directory configs (forms, keys, where they apply, the `json.schemas` editor tip), `--strict` and its parts, `--rule` and patterns, `--out`, `--no-tests`, `vcs.ref`, plugin references (§8.1, `file:`) and the built-in names, TS-only workspace plugin packages (P9), where relative paths in module options are relative to (§9.4), test runners (candidates, `testRunner` pinning, `testRunners`, detection), removed keys and their replacements, the security note (§4).
- `config-schema/` and `define-config/` area READMEs with type-checked samples (D56).
- `-api` `module/` README: marking path and reference options (`OPTION_FORMAT`, §9.4, B5).
- CONTRIBUTE.md glossary: plugin reference, built-in name, config base.
- AGENTS.md trigger table: "a new config key" → schema + sidecar (`npm run schemas`) + CLI README table; "a new built-in plugin" → the built-in table + CLI README.

## Acceptance criteria

- `resolveConfig` produces the documented `ResolvedConfig` for every fixture, and the documented errors for every broken one.
- `dist` contains every `*.d.json.ts` sidecar next to its `.json`; the consumer-style type test over `dist` infers `TsshowcaseConfig` as an object type (C2).
- The published `config.schema.json` validates a sample `tsshowcase.config.json` and rejects an invalid one (C2).
- A TS stub module loaded by path, importing `@ofzza/tsshowcase-api`, runs from a fixture copy without a local install of `-api`.
- `npm run ci` passes.

## Out of scope

Action dispatch, `--help` rendering (C6 supplies `describeFlags`), the report, applying severity overrides and `--strict` (C9, C11); merging directory configs and the other `config/*` issues, applying kinds, exemptions and collections, `tests/no-runner` (C9); listing and running tests (C9, C11, C7); theme selection, duplicate names, `defaultTheme`, scoping, the cache directory itself (C11); calling `scanRepo`/`toRepoInput` (the actions, C9/C11); built-in plugins' own behaviour (C3, C7, C8, C10).

## Recorded for later chunks

- **C9:** dispatches actions and calls `resolveConfig`, then `scanRepo`, `toRepoInput`, `loadDirectoryConfigs` (§4.1) and `loadTestRunners` (§11); adds its action flags (`--format`) and positionals (`init <dir>`) to C6's registry; renders `--help` from `describeFlags`; prints `ConfigError`/`PluginError`; merges `ResolvedConfig.configMaps` with the directory configs per directory (D188) and raises the other `config/*` issues; applies `ResolvedConfig.rules` per source with patterns (D176); `check` honours `strict`'s `warnings` part (D164); places `tests/runner-failed` from `LoadedRunners.issues` (repo-level, D180); turns a module issue emitted under a CLI-reserved namespace into a `module/error` (C2 §9; C6 only rejects declared rules there at load).
- **C11:** uses `ResolvedConfig.outDir` (D167), `tests` (D197) and `dev` (the flags `--out`, `--no-tests`, `--host`, `--port` are C6's registry entries, D111); `dev.*` into `DevContext`; calls `loadTestRunners` in `build`/`dev` and places `LoadedRunners.issues` as C9 does (repo-level, D180); applies `ResolvedConfig.strict` to `build` after writing (D164); `loadThemes()` then duplicate names, `defaultTheme`, scoping; owns `node_modules/.cache/tsshowcase/` (jiti's cache lives under it); the runner set fixed per `dev` session; after every `dev` rescan, re-calls `loadDirectoryConfigs` and restarts the session when the set of existing packages differs from the one C6 resolved (QH).
- **C7:** the built-in runner implements `listTestFiles` (required by §8.2).
- **C8:** `-base-react` declares its `kinds` (keys matching the pattern) and, if any, `testRunners`; its rules outside the reserved namespaces; no `testRunner` option (D172; no migration hint, CC7).
- **C13:** the security note (directory configs execute code too); aliasing of every `@ofzza/tsshowcase*` package for path-loaded plugins (D117); `strictSchema: false` for module schemas; `file:` references (D114); marking path and reference options (D116); directory configs and the `json.schemas` tip; `--strict` parts; `vcs.ref`.
- **C8, C10:** `-base-react`/`react-default` mark their path and reference options (`showcaseStylesheets`, `showcaseSetup`, `viteConfig`) with `OPTION_FORMAT` (D116).
- **C3, C7:** tests for the real built-in names (`default-light`, `default-dark`, `vitest`) (L14).

## C2 amendment B5 (re-approved with this plan, applied)

> History: superseded where the redesign amendment says so (`loadTestRunner` and the `testRunner` option removed, D172).

- **§5.11 Module options:** path-valued options are marked `format: 'tsshowcase-path'`, plugin references `format: 'tsshowcase-reference'`; the CLI makes relative values absolute against the base of the source they were written in (root config, package config, flag) before merging, so modules see absolute paths and absolute `file:` references only (D116). Allowed positions and the combinator restriction as in C6 §9.4. The sample gains a marked `testRunner`.
- **§1 / `module/` unit:** new runtime constant `OPTION_FORMAT = { path: 'tsshowcase-path', reference: 'tsshowcase-reference' } as const`, with a test of its values.
- **§5.7 `loadTestRunner`:** a relative `file:` reference resolves against the repo root (D116); built-in names, npm packages and `file:` references per D114.
- **§10 (wording only, P11):** the schemas live in `config-schema/`, and the CLI's build copies them to the package root for `$schema` paths (C6 §5).

## Super-plan R6 and AGENTS edits (applied)

> History: superseded where the redesign amendment says so (`build [outDir]` → `build --out`, D167; runner loading, D172).

- **R6 (C6 planning, D111–D117, review `06a-c6-plan-review.md`):**
  - C6: "Reference resolution order: built-in name → npm package → path (D9)" → "built-in name → npm package; paths only as `file:` (D9, D114)"; "Config file discovery (`tsshowcase.config.ts|js|json`)" → "`.ts|.mts|.js|.mjs|.json|.jsonc`, JSON with comments (D115)"; adds argv parsing (D111), path-marked module options (D116), aliases for path-loaded plugins (D117), `outDir` resolution.
  - C9: "CLI framework, global flags (`--module`, `--theme`, `--config`, `--cwd`)" → "the command framework and `--help` (flags parsed by C6, D111)".
  - C11: "writes `--out` with the output guard" → "writes the output directory (`build [outDir]`, D93) with the output guard".
  - C2 bullet: amendment B5 (`OPTION_FORMAT`).
- **C1 (wording only, P11):** the CLI package's `files` adds the two schema copies at the package root; its `build` script copies them.
- **AGENTS:** D111–D117 recorded (D111–D115 already in the log); D6 status "refined by D115" and its file list updated; D9 "refined by D114" (done); open points: C6 row removed; C8 and C10 rows gain "mark path/reference options (D116)"; C9 row gains "add action flags and positionals to C6's registry, render help from `describeFlags`"; C11 row gains "`ResolvedConfig.outDir`, `dev.*` → `DevContext`, `loadThemes()`, the jiti cache under the CLI cache directory"; C13 row gains "`file:` references, aliases for path-loaded plugins, marking options, `strictSchema: false`, TS-only workspace plugins"; C6 marked approved in the tracker.

## Redesign amendment (plan 7 of the representation redesign's application; approved 2026-10-21)

Plan 7 of the redesign's application (`08b-representation-redesign.md` §4.2, §8, §9, §12; `08e` E7), after AGENTS, C2, C1, C3, C4 and C5; the choices CC1–CC10, the review's QA–QJ and Fix 1–25 are in `06b-c6-redesign-review.md` (D206–D208). C6 now resolves the redesign's configuration and loads the CLI's own test runners:

- **Goal, §1:** `directory-config.schema.json`, `defineDirectoryConfig`, the `directory-configs/` loader, the argv normaliser; exported areas under C1 §8's dogfood layout.
- **§2:** packages that don't exist are never configured (`packageExists()`, `04b` Q3); module statics checked; `ResolvedConfig` gains `configMaps`, `strict`, `loadTestRunners`, loses `check`, `failOnTestFailure`, `loadTestRunner`; `outDir` from `--out` (D167); `rules` per source (D176); `ResolvedTheme` is the CLI's own type (02b Skip 2).
- **§3:** the argv normaliser (`--strict`, `--no-strict`, `--rule`) and the positional hints (D164, D167, D176).
- **§4.1:** directory configs loaded after the scan, validated, not merged (`04b` J8, CC1).
- **§5:** the redesign's keys, removed keys, root-only keys, the package and directory-config schemas (D164, D165, D172, D176, D188, D189, D191, D192, D205); map keys checked (CC9); the slug pattern (CC10).
- **§6, §7:** `--scope` package-relative and replacing every package's value (C4, `04b` Q2); `--out`, `--strict`, `--rule`, `--no-tests`, `--vcs-ref`; config maps per source (CC4).
- **§8.2, §9.1:** `listTestFiles` required; module `kinds` (no collisions, key pattern, CC5), `testRunners`, rules outside reserved namespaces.
- **§9.4:** `testRunner` no longer a module option.
- **§11:** the CLI's test-runner loading (D172): candidates, pinning, every detecting runner, ids, `tests/runner-failed` (CC2, CC3, CC6).
- **§12:** `scope` per package, `vcs.ref` validation (CC8), the rule-id table (D175) and reserved namespaces.
- **§13:** fixed hints only for plausible mistakes (`exclude` → `skip` in a package config, the `build` positional, `build --strict tests`, top-level `collections`); never-published names (`include`/`exclude` → `scope`/`ignore`, `vcs.defaultBranch`, the removed keys and flags) get none (CC7).
- **§14, §15, out of scope, recorded points:** updated.
- **Elsewhere (wording, plan 7):** C2 §7 `ResolvedTheme` moved into the CLI (02b Skip 2); C2 §5.1 the kind-key pattern (CC5); C3's recorded point on kind keys resolved.
- **Unchanged:** config file formats and discovery (§4; §4.1 added, the security note extended), references and jiti (§8.1, §8.3, §8.4), module options and `optionFlags` (§9.2, §9.3), path-marked options (§9.4) apart from `testRunner`, themes (§10).

## Open questions

- ✅ **Q1 Argv ownership and parser (D111):** C6 owns flag parsing (two-phase `node:util` `parseArgs`) and every option-like flag; C9 owns the command framework and help rendering, declaring its action flags in C6's format (§3).
- ✅ **Q2 Reference syntax (refines D9, D114):** a path reference requires the `file:` prefix (the user's change; relative paths after `file:` resolve against the flag's or config's base, absolute paths and `file:///` URLs accepted; path-like references without it fail with a hint); npm packages resolve from the repo root first, then from the CLI (§8.1).
- ✅ **Q3 Flags vs config arrays (M12, D112):** `--ignore` appends, `--scope` replaces, `--theme` replaces; `--scope`/`--ignore` root-relative *(`--scope` package-relative since D165, redesign amendment)*; path references relative to the process cwd for flags and to the config base in configs (§6).
- **Q4 Config file formats:** ✅ `.ts`, `.mts`, `.js`, `.mjs`, `.json`, `.jsonc`, JSON with comments (D115, the user's change). ✅ Several config files in one directory is an error (D115). ✅ No function-export form (D115): a `.ts`/`.js` config default-exports an object (typically wrapped in `defineConfig`, D66, unchanged); every action needs the same settings to work together, and the form can be added later, additively (§4).
  - ✅ **Q4a (D115):** no trailing commas.
  - ✅ **Q4b (D115):** `tsshowcase.config.jsonc` accepted, parsed like `.json`.
- ✅ **Q5 `dev` keys (K8, D113):** `dev: { host, port }` with `--host` / `--port`, defaults left to the module/toolchain (§5, §6).
