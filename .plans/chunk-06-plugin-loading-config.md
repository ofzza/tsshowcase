# Chunk plan C6: Plugin loading + configuration (CLI core)

> Status: **APPROVED** (round 4, 2026-10-19). Open questions Q1–Q5 resolved (D111–D115); the fresh-eyes review `06a-c6-plan-review.md` (P1–P14) applied (D116, D117). Approved together with C2 amendment B5 and super-plan R6, both applied. **C6 amendment (C5 planning) approved 2026-10-19 with the C5 plan:** `docs/*` rule ids (§12). Decisions referenced as `D<n>` are in AGENTS.md §3.

## Goal

Turn the command line, the root config file and the per-package config files into one **resolved configuration**, and load the plugins it names: the rendering module (instantiated, with validated options), theme sets (resolved and version-checked; C11 selects and scopes, K14) and test runners (on the module's behalf, through `ctx.loadTestRunner`, D34, D74). Produce C4's scanner inputs from the config. Ship the user-facing config schema, its inferred types and `defineConfig` (D66).

C6 does not run any action. C9 (`check`, `init`) and C11 (`build`, `dev`) call it.

## 1. Where the code lives

```
packages/tsshowcase/src/
  config-schema/    exported unit: config.schema.json, package-config.schema.json (+ tsschema sidecars, D62), configSchema, packageConfigSchema, TsshowcaseConfig, PackageConfig
  define-config/    exported unit: defineConfig(), definePackageConfig() (P6)
  args/             internal: flag tables, the two-phase argv parse (§3)
  config/           internal: discovery, loading, validation, error formatting, merging, scanner inputs (§4–§7, §12)
  plugins/          internal: reference resolution (D9), jiti loading, version checks, module / theme / runner loading, loadTestRunner (§8–§11)
  resolve-config/   internal: resolveConfig(), ties it together (§2)
```

The two exported units follow D41 (own directory, README, tests). The internal directories follow D10 and have tests, like C4's `scan/`.

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

Every action's flags and positionals live in **one static registry** in `args/` (§3, P3): the CLI's own option flags (C6), plus the action-only flags and positionals that C9 and C11 add to it (`check --format`, `dev --host`/`--port`, `build [outDir]`, `init <dir>`). `resolveConfig` reads the current action's entries from it; collisions are checked against all of it.

Order inside `resolveConfig`:

1. **Phase-1 argv parse** (§3): CLI flags only; module flags are kept as unparsed tokens.
2. **Root:** `--cwd`, else the process cwd; no upward search (C4 §3, L4). Must be an existing directory.
3. **Root config:** discover or take `--config` (§4), load, validate against `config.schema.json` (a `skip` key is held back for step 4).
4. **`detectWorkspace(root)`** (C4 §2); then the L4 check for a `skip` key in the root config (§4, P12).
5. **Per-package configs** in every `WorkspaceInfo.packages` directory (§7), validated against `package-config.schema.json`.
6. **Module:** resolve the reference (flag → config → `react-default`), load, version-check (§8, §9). Not yet instantiated.
7. **Phase-2 argv parse** with the module's `optionFlags` added (§3, §9.2); any unknown flag or extra positional is now an error.
8. **Options:** path-marked values in each source made absolute against that source's base (§9.4, D116); then root `moduleOptions` + flags → validate → `ModuleInit.options`; per package: root `moduleOptions` shallow-merged with the package's own + flags → validate → `PackageInput.options` (D69).
9. **Scan inputs:** `scope` (validated with C4's `validateScope()`), `ignore`, `skippedPackages`, `outDir` (positional > config > default, D93; only an explicit one goes to the scanner, validated early, M13, P2), `vcs` overrides (§12).
10. **Rule ids:** `rules` keys checked against the module's `rules`, plus the CLI-known `scan/*`, `docs/*` and `module/error` (M15).
11. **Instantiate the module** with `ModuleInit` (§9.3).

Themes are loaded on demand (`resolved.loadThemes()`, §10): only `build` and `dev` need them, and a broken theme reference must not fail `check`. Test runners are loaded on demand by the module (§11).

After `resolveConfig` the action calls `scanRepo(resolved.scan)` and `toRepoInput(scan, resolved.packageOptions)` (C4 §2), as recorded in C4.

```ts
export interface ResolvedConfig {
  readonly root: string; // absolute
  readonly configFile?: string; // absolute; undefined when none was found
  readonly workspace: WorkspaceInfo; // C4
  readonly module: LoadedModule; // §9: reference, resolved URL, constructor, instance
  readonly moduleOptions: unknown; // validated root options (ModuleInit.options)
  readonly packageOptions: ReadonlyMap<RepoPath, unknown>; // validated per-package options, for toRepoInput (D69)
  readonly rules: ResolvedRules; // root rules + per-package rules (C9 applies them, D70)
  readonly scan: Omit<ScanOptions, "logger">; // C4 ScanOptions; the action adds its logger
  readonly themeReferences: readonly ThemeReference[]; // §10, already parsed; loaded by loadThemes()
  readonly defaultTheme?: string;
  readonly outDir: { readonly path: string; readonly explicit: boolean }; // absolute; positional > config > default ./showcase relative to the cwd (D93, §6, P2); only `explicit` ones reach the scanner (D101)
  readonly check: boolean; // D76
  readonly tests: boolean;
  readonly failOnTestFailure: boolean; // D8
  readonly dev: { readonly host?: string; readonly port?: number }; // K8
  readonly actionFlags: { readonly [name: string]: unknown }; // parsed values of the action-only flags in the registry (e.g. `format`)
  readonly positionals: { readonly [name: string]: string | undefined }; // by registry name, e.g. { dir: 'src/Button' } for `init <dir>`; `outDir` is resolved into `outDir` above
  loadThemes(): Promise<readonly ResolvedTheme[]>; // §10
  loadTestRunner(repo: RepoInput): TestContext["loadTestRunner"]; // §11: returns the bound helper for TestContext
  pluginHost(name: string): PluginHost; // for any plugin the CLI constructs (prefixed logger, versions)
}
```

## 3. Argv parsing (Q1)

**Ownership (D111):** C6 owns the **flag machinery and every option-like flag** (the ones that map onto config keys, plus module flags, D79) and parses positionals; C9 owns the command framework: action dispatch, `--help`/`--version` rendering, the report. C9 and C11 declare their action-only flags (`--format`, `--host`, …) and positionals (`init <dir>`, `build [outDir]`) as entries in C6's static registry (§2), so there is one parser and one collision check.

- **Parser:** `node:util` `parseArgs` (no dependency), in **two phases**, because module flags are only known once the module is loaded (D79) and the module reference may itself come from `--module` or from the config: phase 1 parses the CLI's flags with `strict: false` and `tokens: true`, keeping unknown tokens; phase 2 parses again with the module's flags added, `strict: true`, `allowPositionals: true`, and the action's positional arity checked (P2).
- **Registry entries:** flags `{ name, type: 'boolean' | 'string' | 'number', multiple?, negatable?, description, configKey?, actions }` and positionals `{ name, action, required, description, configKey? }`. `negatable` booleans accept `--no-<name>`; numbers are coerced and validated; `multiple` flags repeat (`--scope a --scope b`).
- **Help (P3):** `describeFlags(action, argv, cwd)` returns the effective list (CLI, action and module flags and positionals, each with its description) for C9's `--help`. It never throws: it loads the config and the module only as far as needed for the module's `optionFlags` (no options validation, no instantiation), and when that fails it returns the CLI's own entries plus a warning saying why the module's flags are missing. `--help` short-circuits `resolveConfig`.
- **Collisions (D79, P3):** a module flag is checked against **every** entry of the registry, for all actions (module flags are accepted on every action, so a clash is the same error whichever action runs), and against `help` and `version`. A module flag may not start with `no-`; the `no-` form of a negatable flag is reserved with it. The CLI also reserves the `vcs-` prefix. A clash is a load-time error naming the module.

## 4. Config files

- **Names (Q4):** `tsshowcase.config.ts`, `.mts`, `.js`, `.mjs`, `.json`, `.jsonc` (comments allowed in both, below; D115), looked up in the root only (no upward search, L4). **Several present is an error** naming them (no precedence to guess). None present is fine: everything has a default.
- **`--config <path>`** (relative to the process cwd) loads that file instead; it must exist. Its directory becomes the **config base** for config-relative values (§6). The root stays `--cwd` / the process cwd.
- **Loading:** `.json` and `.jsonc` are parsed the same way (D115): `//` and `/* */` comments are allowed and ignored (parsed with Microsoft's `jsonc-parser`, the parser VS Code uses; syntax errors are reported with line and column); everything else is strict JSON, **trailing commas rejected**; `$schema` is allowed. `.jsonc` gets comment highlighting in editors out of the box; for a `.json` file with comments the CLI README shows the VS Code `files.associations` line; `.ts`/`.mts`/`.js`/`.mjs` through jiti (§8.3), taking the **default export**, which must be a plain object (or a promise of one), typically wrapped in `defineConfig` (D66). A default export that is a function is rejected with a clear error: every action needs the same settings to work together, so there is no per-action config (D115; it can be added later, D13).
- **Aliases for config files (L12, D117):** the same aliases as for path-loaded plugins (§8.3): every `@ofzza/tsshowcase*` package resolves to the running CLI's own copy, so `import { defineConfig } from '@ofzza/tsshowcase'` works under `npx` and pnpm without a local install.
- **Validation:** ajv against `config.schema.json` (§5), `allErrors: true`, **no `useDefaults`** for the config: defaults are applied in code, because "explicitly configured" matters (an explicit `outDir` is a hard ignore, D101, while the default `./showcase` is not).
- **A package config at the root (L4, P12):** checked after `detectWorkspace` (§2 step 4). A root config containing `skip` gets a dedicated error: "`skip` is only valid in a workspace package's config"; when C4 detected that the root is a package of a parent workspace, the hint adds "run tsshowcase from the workspace root at `<path>`".
- **Security note (documented, C13):** `.ts`/`.js` configs and path-loaded plugins execute code; running `check` on an untrusted PR runs that code.

## 5. The config schema (D58, D62, D66, C2 §10)

Two plain JSON Schema files with committed tsschema sidecars, in the `config-schema/` unit. `additionalProperties: false` everywhere. Documentation in each property's `description` (shown by editors via `$schema`). Defaults are stated in the descriptions, not as `default` keywords (§4).

**Root config (`config.schema.json`):**

| Key                 | Type                                                                                           | Default (applied in code)               | Notes                                                                  |
| ------------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------- | ---------------------------------------------------------------------- |
| `$schema`           | string                                                                                         | –                                       | ignored                                                                |
| `module`            | string                                                                                         | `'react-default'`                       | reference (D9, §8)                                                     |
| `moduleOptions`     | object                                                                                         | `{}`                                    | validated by the module's `optionsSchema` (§9)                         |
| `themes`            | string \| string[] (min 1)                                                                     | `['default-light', 'default-dark']`     | references, optional `#name` (§10)                                     |
| `defaultTheme`      | string                                                                                         | –                                       | a theme name; checked by C11                                           |
| `outDir`            | string                                                                                         | `./showcase` (relative to the cwd, D93) | relative to the config base; an explicit value is a hard ignore (D101) |
| `scope`             | string[]                                                                                       | `['.']`                                 | root-anchored globs (D107, C4 §5)                                      |
| `ignore`            | string[]                                                                                       | `[]`                                    | gitignore syntax incl. `!` (D103, D107, C4 §4)                         |
| `rules`             | object of `'error' \| 'warning' \| 'off'`                                                      | `{}`                                    | D70                                                                    |
| `testRunners`       | string[]                                                                                       | `[]`                                    | extra auto-detection candidates (D42, §11)                             |
| `check`             | boolean                                                                                        | `true`                                  | D76                                                                    |
| `tests`             | boolean                                                                                        | `true`                                  | `--no-tests`                                                           |
| `failOnTestFailure` | boolean                                                                                        | `false`                                 | D8                                                                     |
| `vcs`               | object: `webUrl`, `directory`, `host` (`github` \| `gitlab` \| `bitbucket`), `fileUrlTemplate` | –                                       | D110, C4 §9                                                            |
| `dev`               | object: `host` (string), `port` (integer 0–65535)                                              | module/toolchain default                | K8 (Q5)                                                                |

**Package config (`package-config.schema.json`):** `$schema`, `skip` (boolean), `rules`, `moduleOptions` (D59, D69). Any root-only key there gets a dedicated error: "`themes` can only be set in the root config (`<path>`)".

**Inferred types:** `TsshowcaseConfig` and `PackageConfig` from the sidecars (C2 §10), never hand-written; `rules`/`moduleOptions` infer as `{ [key: string]: unknown }` until tsschema infers records (D58, the known gap before `1.0.0`). `defineConfig(config: TsshowcaseConfig): TsshowcaseConfig` and `definePackageConfig(config: PackageConfig): PackageConfig` (P6, for typed package configs) are identities.

**Publishing (C2 §10, D62):** the schema files and their sidecars are emitted into `dist` (C1 A1). The README documents `"$schema": "./node_modules/@ofzza/tsshowcase/config.schema.json"`, which needs a real file at the package root: the CLI package's own `build` script copies both schema files to the package root after `tsc` (gitignored, listed in `files` next to `dist`). The `exports` subpaths `./config.schema.json` and `./package-config.schema.json` have a `source` condition pointing at `src/config-schema/*.json` (so in-repo imports work without a build, C1 §3) and a `default` pointing at the root copies (P11). This deviates from C2 §10 ("no emit step", a `config/` directory) and C1's `files: ["dist"]`; both get wording-only notes (R6).

## 6. Precedence and path bases (M12, Q3)

**Precedence:** built-in defaults < root config < package config (package keys only) < flags (D6, D79).

| Flag                                                                        | Config key             | Arrays       | Paths relative to                                                                                            |
| --------------------------------------------------------------------------- | ---------------------- | ------------ | ------------------------------------------------------------------------------------------------------------ |
| `--cwd <dir>`                                                               | –                      | –            | process cwd                                                                                                  |
| `--config <file>`                                                           | –                      | –            | process cwd                                                                                                  |
| `--module <ref>`                                                            | `module`               | –            | `file:` references (§8.1): flag: process cwd; config: config base                                            |
| `--theme <ref>` (repeatable)                                                | `themes`               | **replaces** | as `--module`                                                                                                |
| `--default-theme <name>`                                                    | `defaultTheme`         | –            | –                                                                                                            |
| `--scope <glob>` (repeatable)                                               | `scope`                | **replaces** | repo root (both)                                                                                             |
| `--ignore <pattern>` (repeatable)                                           | `ignore`               | **appends**  | repo root (both)                                                                                             |
| `--vcs-web-url`, `--vcs-directory`, `--vcs-host`, `--vcs-file-url-template` | `vcs.*`                | –            | –                                                                                                            |
| `--no-check` (build)                                                        | `check`                | –            | –                                                                                                            |
| `--no-tests` (build, dev)                                                   | `tests`                | –            | –                                                                                                            |
| `--fail-on-test-failure` (build)                                            | `failOnTestFailure`    | –            | –                                                                                                            |
| `--host`, `--port` (dev)                                                    | `dev.host`, `dev.port` | –            | –                                                                                                            |
| `build [outDir]` (positional, declared by C11, resolved by C6, P2)          | `outDir`               | –            | positional: process cwd; config: config base (D93)                                                           |
| module flags (`optionFlags`, D79)                                           | `moduleOptions.*`      | per schema   | path-marked options: flag: process cwd; root config: config base; package config: its directory (§9.4, D116) |

Rationale for the arrays (M12): `--ignore` appends because the common use is "this run, also leave out X"; `--scope` replaces because the common use is "only look at X". Both stay root-relative, like their config keys, so a pattern means the same wherever it is written.

`--theme` replaces, so `--theme default-dark` gives a single-theme build.

## 7. Per-package configs (D59, D69)

- **Discovery:** for `layout: 'workspace'`, the same six file names in each `WorkspaceInfo.packages` directory; several in one package is an error. Loaded like the root config (jiti, aliases), validated against the package schema. Config files are metadata: read whether or not the package is in `scope` or ignored (D101).
- **`skip: true`** puts the package into `skippedPackages` (C4: a hard cut, nested packages included).
- **Merging:** path-marked option values are first made absolute against each source's own base (§9.4, D116); then `rules` and `moduleOptions` are **shallow-merged** over the root values (C2 §10); module flags are then applied on top of every merged result, then each result is validated against `optionsSchema` (D69, D79). A validation error names the package config file it came from.
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

Each reference is loaded for a known **kind** (module, theme set, runner). The default export must be a class (a function with a `prototype`); its static `apiVersion` must be a valid semver range (`semver.validRange`), and `semver.satisfies(IMPLEMENTED_API_VERSION, apiVersion, { includePrerelease: true })` must hold (D13, D80; prereleases such as `0.2.0-next.0` satisfy `^0.2`, P7). After construction the instance is shape-checked for its kind (module: `analyze` is a function and every optional method that is present is a function; theme set: `themes` array; runner: `detect`, `run`, `watch`). Errors name the reference, the resolved file and what to do, e.g. "module `file:./tools/my-module.ts` targets tsshowcase API `^0.2`, but this CLI implements `0.1.3`: upgrade `@ofzza/tsshowcase` or use a module version written for 0.1".

### 8.3 jiti and the CLI's packages for path-loaded plugins (L12, D117)

- **Path references** (TS or JS) load through **jiti** (one instance per invocation, ESM interop, `interopDefault`), so custom plugin authors need no build step (D9). jiti **aliases every `@ofzza/tsshowcase*` package in the CLI's dependency tree** (`-api`, `-base-react`, `-base-vite`, `-theme-default`, `-testrunner-vitest`, `-module-react-default`, the CLI itself) **to the CLI's own copies** (P4, D117). A path-loaded plugin therefore works under `npx`, pnpm and when only the CLI is installed; a custom layout module extending `-base-react`'s base class (D48) gets the base and `-api` from the same tree, never two contract copies. Other npm dependencies of a path-loaded plugin resolve normally from its own location. Documented for authors (C13).
- **npm packages and built-ins** load with native `import()` of the resolved file: they are published JS and keep their own dependencies (whose ranges npm resolved); no aliasing.
- jiti's transpile cache goes to the CLI cache directory (`<root>/node_modules/.cache/tsshowcase/jiti`, C11 owns the directory) **only when `<root>/node_modules` exists**; otherwise jiti's own default in the OS temp directory, so the CLI never creates `node_modules/` in a repo that has none (P10). `dev` doesn't reload plugins or configs: a config change restarts (C4's recorded C11 point).

### 8.4 `PluginHost`

Every constructed plugin receives `{ apiVersion: IMPLEMENTED_API_VERSION, cliVersion, logger }`, the logger prefixed with the plugin's display name (`displayName`, else the reference).

## 9. The module

### 9.1 Loading

`LoadedModule = { reference, url, constructor: ShowcaseModuleConstructor, instance: ShowcaseModule }`. The static members `optionsSchema`, `rules` and `optionFlags` are validated at load: `optionsSchema` must compile with ajv (a compile error is a module error), `rules` ids must be unique, every `optionFlags` target must be a supported schema path (§9.2).

### 9.2 Module options and `optionFlags` (D79)

- **ajv** (`ajv/dist/2020`), `allErrors: true`, `useDefaults: true` (C2 §5.11: defaulted properties are required in `OptionsOf`), `strictSchema: false` for module schemas (third-party schemas may carry editor keywords), `strict: true` for our own. Options are deep-cloned before validation (`useDefaults` mutates). No schema: options pass through as given.
- **Flags from the schema:** the target path is followed through `properties` (and local `$ref`s); the property's type decides the flag: `boolean` → `--x` / `--no-x`; `number`/`integer` → coerced; `string` (incl. `enum`, validated by ajv) → value; an array of those → repeatable. Objects, unions and `oneOf`/`anyOf` targets are a load-time error naming the module and the flag. The schema `description` becomes the help text. Flag names must be kebab-case.
- Flags apply to the root options and to every merged per-package result (§7), then validation runs.

### 9.3 Instantiation

After options are validated: `new Module({ ...pluginHost, options, repoRoot })`. A throwing constructor is a module error naming the module.

### 9.4 Path-valued options (D116, P1, C2 amendment B5)

Module options may hold paths (`showcaseStylesheets`, `showcaseSetup`, C3 §1) and plugin references (`testRunner`, D34). After root config, package configs and flags are merged, nobody could tell which base a relative value was written against, so **modules mark them in their schema** and the CLI resolves them **per source, before merging**:

- **`format: 'tsshowcase-path'`** (constant `OPTION_FORMAT.path` in `-api`, B5) on a string property: the value is a filesystem path; relative values are made absolute against the source's base (root config: the config base; package config: its directory; flags: the process cwd, D112).
- **`format: 'tsshowcase-reference'`** (`OPTION_FORMAT.reference`): the value is a plugin reference (§8.1); only `file:` values are rewritten (`file:./x.ts` → `file:/abs/x.ts`), built-in names and npm packages pass unchanged.
- The marker may sit on a property, on array `items` or on `additionalProperties`, reached through `properties`/`items`/`additionalProperties` and local `$ref`s. A marker under `oneOf`/`anyOf`/`allOf`/conditionals is a load-time error naming the module (the CLI couldn't tell which branch applies). ajv registers both formats (any string is valid; the marker only drives the rewrite).
- **Modules therefore only ever see absolute paths** (and absolute `file:` references), so `ctx.loadTestRunner(this.options.testRunner)` needs no base. A relative `file:` reference passed to `loadTestRunner` anyway (an unmarked option, a hard-coded value) resolves against the repo root, with a debug log line.
- Absolute paths in options never reach the site by themselves; a module that embeds an option value in its output keeps C2 rule 2 (no leaks) itself.
- Documented in the `-api` `module/` README (how to mark options) and the CLI README (where relative paths in module options are relative to). Our own modules mark theirs (C8, C10).

## 10. Themes (K14: C6's half)

- **Parsing:** each `themes` entry is `<ref>` or `<ref>#<name>` (C2 §7); the built-in names are aliases for `@ofzza/tsshowcase-theme-default#<name>`. Parsed in `resolveConfig`, so syntax errors fail early; `ThemeReference = { reference, base, name? }`.
- **`loadThemes()`** (build and dev): loads every referenced theme set once (two references to the same set share an instance), version-checks it, validates each `ThemeDefinition`: `name` matches `^[A-Za-z0-9][A-Za-z0-9_-]*$` (it lands in an attribute selector and in `localStorage`), `scheme` is `light` or `dark`, `css` is a non-empty array of absolute paths to existing `.css` files. `<ref>#<name>` must name a theme in the set (the error lists the available names). Returns `ResolvedTheme[]` in config order (`reference` set).
- **Not here (C11):** duplicate names across references, `defaultTheme` existing, scoping, the cache.

## 11. `loadTestRunner(ref?, { candidates? })` (D42, D74)

`resolved.loadTestRunner(repo)` returns the helper the CLI puts on `TestContext`:

- **With `ref`:** resolve (§8, kind runner), version-check, construct; failure throws. `ref` need not be among the candidates.
- **Without `ref`:** the candidate set is the built-ins (`vitest`) plus config `testRunners`, intersected with `options.candidates` when given. **Candidates are deduplicated by identity (P8):** built-in names are first mapped to their package (`vitest` → `@ofzza/tsshowcase-testrunner-vitest`), and npm references compare by package name plus export subpath, `file:` references by absolute path; so `'vitest'`, `'@ofzza/tsshowcase-testrunner-vitest'` and a copy of that package installed in the repo are one candidate (resolved as the built-in, from the CLI). Each candidate's `detect(runnerRepo)` runs (`RunnerRepoInput` from the `RepoInput`). Exactly one match → returned and logged; several → throws "several test runners detected (`vitest`, `file:./tools/jest-runner.ts`); set the module option `testRunner`"; none → `undefined`. A candidate that fails to load or whose `detect` throws is an error naming it (not silently skipped).
- Instances are cached per identity for the invocation.

## 12. Scanner inputs, VCS and rules

- **`scope`:** `validateScope()` from C4 (absolute paths and `..` rejected) on config values and flags.
- **`ignore`:** passed through (gitignore syntax is validated by C4's matcher; an unparseable pattern names its source: the config file or `--ignore`).
- **`skippedPackages`:** from package configs (§7).
- **`outDir` (M13, P2):** resolved here: the `build` positional, else config `outDir`, else `./showcase` relative to the cwd (D93). Only an **explicit** value (positional, config) is passed to the scanner (D101). It is validated early with the same check `scanRepo` uses (C4 §10: not the root, not an ancestor of it, not containing a package), shared as one helper exported internally from `scan/`.
- **`vcs`:** the four keys and flags merged (flag → config) into C4's `VcsOverrides`; `vcs.host` must be one of the three hosts; `fileUrlTemplate` must contain `{path}`.
- **Rule ids (M15):** root and package `rules` keys are checked against the module's `rules` plus the CLI-known ids (`scan/no-manifest`, `scan/invalid-manifest`, `scan/scope-unmatched`, `scan/unreadable`, `docs/broken-link`, `docs/broken-anchor`, `module/error`; the `docs/*` ids from C5, D123). An unknown id in the CLI's own namespaces (`scan/`, `docs/`, `module/`) always gives a warning naming the file; an unknown id in any other namespace warns only when the module declares `rules` (C2 §9), since without them the CLI can't know the module's ids (P14).

## 13. Errors

- **`ConfigError`** (thrown; C9 prints it and exits non-zero): the file (or `flag --x`), the JSON path (`moduleOptions.testRunner`), a message and an optional hint. ajv errors are reformatted (one line per error, deduplicated, `additionalProperties` errors name the property).
- **"Did you mean":** an unknown key gets the closest known key (edit distance ≤ 2), plus fixed hints for renamed keys: `include` → `scope`, `exclude` → `ignore` (D107), `vcs.defaultBranch` → removed, links use `HEAD` (D110), `exclude` in a package config → `skip` (D69).
- **`PluginError`:** a plugin that fails to resolve, load, version-check, validate or construct; names the kind, the reference and the resolved file. Shared with C9/C11 for later plugin failures (`module/error`).

## 14. Fixtures and tests

**Stub plugins (L14):** C6's tests never depend on the built-in packages' contents (C3 and C7 are implemented later): built-in name lookups are tested with the table injected. The real built-in names get their tests with C3 and C7.

| Fixture                                                  | Purpose                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `plugins-stub`                                           | Stub plugins in TS and JS importing `@ofzza/tsshowcase-api` (the alias): a module with `optionsSchema`, `rules` and `optionFlags`; a module without a schema; a theme set with two themes and CSS files; runners whose `detect` returns true / false / throws; a module with path- and reference-marked options (incl. arrays and a `$ref`) and one with a marker under `oneOf`; a layout module extending a stub base package aliased like `-base-react`; failure stubs (non-class default export, invalid range, unsatisfied range, missing `analyze`, a colliding flag (incl. one colliding only with another action's flag, and a `no-` name), an unsupported flag target, a throwing constructor) |
| `config-ts`, `config-json`, `config-jsonc`, `config-mjs` | One config per format, each using `defineConfig` or `$schema`, pointing at the stub plugins                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `config-several`                                         | Two config files (error)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `config-workspace`                                       | A workspace with package configs: `skip`, `rules`, `moduleOptions` (incl. relative path-marked values), one with a root-only key (error), a `tsshowcase.config.ts` using `definePackageConfig`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `config-package-at-root`                                 | A config with `skip` at the root: the L4 error standalone and, run on a package of a workspace, with the workspace-root hint (P12)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

npm-installed stubs (a plugin in `node_modules`, an `exports`-only ESM package, one installed next to the CLI only, a copy of a built-in runner package installed in the repo, P8) are created at test time with `copyFixture`'s `files` option.

Tests, per directory of §1:

- argv: both phases, unknown flags and extra positionals in each phase, positional arity, repeatable and negatable flags, number coercion, collisions against every action's flags, `help`/`version`, `no-` names and the `vcs-` prefix (the same result under every action), `describeFlags` with a loadable module, with invalid `moduleOptions` (still lists the module's flags) and with an unloadable module (CLI flags plus the warning);
- config files: each format, JSON and JSONC with `//` and `/* */` comments (incl. comment-like text inside strings kept), trailing commas rejected, JSON syntax errors with line and column, `--config`, several files, none, default export checks, the aliases (a config importing `defineConfig` from a copy without a local install), validation errors and their formatting, every "did you mean" hint, the L4 error;
- precedence and bases: every row of §6, `--ignore` appending and `--scope` replacing, `outDir` positional > config > default and explicit vs default, `outDir` early validation; path- and reference-marked options from the root config, a `--config` elsewhere, a package config and a flag each resolved against their own base before merging, unchanged built-in and npm references, a relative `file:` passed straight to `loadTestRunner`, a marker under `oneOf` rejected;
- package configs: discovery, several in one package, `skip`, shallow merge, flags on top, per-package validation errors naming the file, root-only keys;
- references: every `file:` form (relative with and without `./`, absolute, `file:///`, a Windows drive, a directory, a theme `#name` after a path), built-in table, resolution from the repo then the CLI, `exports`-only ESM packages, the missing-prefix hint for every path-like form (never loaded), Windows-style paths normalised;
- loading: version checks (valid/invalid/unsatisfied ranges, `^0.1` vs `0.2.0`, a prerelease `0.2.0-next.0` against `^0.2`), shape checks, constructor errors, `PluginHost` contents, the aliases for path-loaded plugins (`-api` and a base package resolve to the CLI's copies) and **no** alias for npm-loaded ones; the jiti cache in the repo only when `node_modules` exists;
- module options: defaults applied, validation errors, no schema, `optionFlags` for every supported type, unsupported targets, flags on root and package options;
- themes: parsing, `#name` selection and its error, shared instances, definition validation, version checks;
- `loadTestRunner`: explicit ref, one / several / no match, `candidates` intersection by identity, a built-in runner also installed in the repo counted once, a throwing `detect`, caching;
- rule ids: module ids, CLI-known ids, unknown ids in `scan/`/`docs/`/`module/` always warned, others only with module `rules`;
- the exported units: `defineConfig` and `definePackageConfig` (type tests: a config object type-checks, a wrong key fails, `skip` only in a package config), `configSchema` validates a sample config and rejects an invalid one (C2 acceptance), a consumer-style type test over `dist` infers `TsshowcaseConfig` as an object type, not `never` (C2 acceptance, D62), the package-root schema copies match the sources, the `source` export condition resolves the schemas without a build.

## 15. Documentation

- CLI README: the config file (formats, discovery, `--config`, `$schema`, `defineConfig`), every key with default and flag (§5, §6), precedence and path bases, the `scope` (root-anchored globs) vs `ignore` (gitignore syntax) asymmetry (N16), per-package configs, plugin references (§8.1, `file:`) and the built-in names, TS-only workspace plugin packages (P9), where relative paths in module options are relative to (§9.4), `testRunners` and auto-detection, the security note (§4).
- `config-schema/` and `define-config/` unit READMEs with type-checked samples (D56).
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

Action dispatch, `--help` rendering (C6 supplies `describeFlags`), the report, applying severity overrides (C9); theme selection, duplicate names, `defaultTheme`, scoping, the cache directory itself (C11); calling `scanRepo`/`toRepoInput` (the actions, C9/C11); built-in plugins' own behaviour (C3, C7, C8, C10).

## Recorded for later chunks

- **C9:** dispatches actions and calls `resolveConfig`; adds its action flags (`--format`) and positionals (`init <dir>`) to C6's registry; renders `--help` from `describeFlags`; prints `ConfigError`/`PluginError`; applies `ResolvedConfig.rules` (D70).
- **C11:** adds `build [outDir]`, `--host`/`--port`, `--no-check`, `--no-tests`, `--fail-on-test-failure` to the registry and uses `ResolvedConfig.outDir` (resolved by C6); `dev.*` into `DevContext`; `loadThemes()` then duplicate names, `defaultTheme`, scoping; owns `node_modules/.cache/tsshowcase/` (jiti's cache lives under it).
- **C13:** the security note; aliasing of every `@ofzza/tsshowcase*` package for path-loaded plugins (D117); `strictSchema: false` for module schemas; `file:` references (D114); marking path and reference options (D116).
- **C8, C10:** `-base-react`/`react-default` mark their path and reference options (`testRunner`, `showcaseStylesheets`, `showcaseSetup`) with `OPTION_FORMAT` (D116).
- **C3, C7:** tests for the real built-in names (`default-light`, `default-dark`, `vitest`) (L14).

## C2 amendment B5 (re-approved with this plan, applied)

- **§5.11 Module options:** path-valued options are marked `format: 'tsshowcase-path'`, plugin references `format: 'tsshowcase-reference'`; the CLI makes relative values absolute against the base of the source they were written in (root config, package config, flag) before merging, so modules see absolute paths and absolute `file:` references only (D116). Allowed positions and the combinator restriction as in C6 §9.4. The sample gains a marked `testRunner`.
- **§1 / `module/` unit:** new runtime constant `OPTION_FORMAT = { path: 'tsshowcase-path', reference: 'tsshowcase-reference' } as const`, with a test of its values.
- **§5.7 `loadTestRunner`:** a relative `file:` reference resolves against the repo root (D116); built-in names, npm packages and `file:` references per D114.
- **§10 (wording only, P11):** the schemas live in `config-schema/`, and the CLI's build copies them to the package root for `$schema` paths (C6 §5).

## Super-plan R6 and AGENTS edits (applied)

- **R6 (C6 planning, D111–D117, review `06a-c6-plan-review.md`):**
  - C6: "Reference resolution order: built-in name → npm package → path (D9)" → "built-in name → npm package; paths only as `file:` (D9, D114)"; "Config file discovery (`tsshowcase.config.ts|js|json`)" → "`.ts|.mts|.js|.mjs|.json|.jsonc`, JSON with comments (D115)"; adds argv parsing (D111), path-marked module options (D116), aliases for path-loaded plugins (D117), `outDir` resolution.
  - C9: "CLI framework, global flags (`--module`, `--theme`, `--config`, `--cwd`)" → "the command framework and `--help` (flags parsed by C6, D111)".
  - C11: "writes `--out` with the output guard" → "writes the output directory (`build [outDir]`, D93) with the output guard".
  - C2 bullet: amendment B5 (`OPTION_FORMAT`).
- **C1 (wording only, P11):** the CLI package's `files` adds the two schema copies at the package root; its `build` script copies them.
- **AGENTS:** D111–D117 recorded (D111–D115 already in the log); D6 status "refined by D115" and its file list updated; D9 "refined by D114" (done); open points: C6 row removed; C8 and C10 rows gain "mark path/reference options (D116)"; C9 row gains "add action flags and positionals to C6's registry, render help from `describeFlags`"; C11 row gains "`ResolvedConfig.outDir`, `dev.*` → `DevContext`, `loadThemes()`, the jiti cache under the CLI cache directory"; C13 row gains "`file:` references, aliases for path-loaded plugins, marking options, `strictSchema: false`, TS-only workspace plugins"; C6 marked approved in the tracker.

## Open questions

- ✅ **Q1 Argv ownership and parser (D111):** C6 owns flag parsing (two-phase `node:util` `parseArgs`) and every option-like flag; C9 owns the command framework and help rendering, declaring its action flags in C6's format (§3).
- ✅ **Q2 Reference syntax (refines D9, D114):** a path reference requires the `file:` prefix (the user's change; relative paths after `file:` resolve against the flag's or config's base, absolute paths and `file:///` URLs accepted; path-like references without it fail with a hint); npm packages resolve from the repo root first, then from the CLI (§8.1).
- ✅ **Q3 Flags vs config arrays (M12, D112):** `--ignore` appends, `--scope` replaces, `--theme` replaces; `--scope`/`--ignore` root-relative; path references relative to the process cwd for flags and to the config base in configs (§6).
- **Q4 Config file formats:** ✅ `.ts`, `.mts`, `.js`, `.mjs`, `.json`, `.jsonc`, JSON with comments (D115, the user's change). ✅ Several config files in one directory is an error (D115). ✅ No function-export form (D115): a `.ts`/`.js` config default-exports an object (typically wrapped in `defineConfig`, D66, unchanged); every action needs the same settings to work together, and the form can be added later, additively (§4).
  - ✅ **Q4a (D115):** no trailing commas.
  - ✅ **Q4b (D115):** `tsshowcase.config.jsonc` accepted, parsed like `.json`.
- ✅ **Q5 `dev` keys (K8, D113):** `dev: { host, port }` with `--host` / `--port`, defaults left to the module/toolchain (§5, §6).
