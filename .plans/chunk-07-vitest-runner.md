# Chunk plan C7: Vitest test runner (`@ofzza/tsshowcase-testrunner-vitest`)

> Status: **APPROVED** (round 3, 2026-10-19), together with C2 amendment B7, the C4 and C6 amendments and super-plan R8, all applied. Open questions Q1–Q6 resolved (D132–D137). Fresh-eyes review (`07a-c7-plan-review.md`, 31 findings, checked against the Vitest 3.2.7, 4.1.12 and 5.0.5 sources) and two spikes (nested Vitest, nested npm installs) applied. Amendments to C2 (B7), C4, C6 and C8, and super-plan R8, are listed at the end (applied). Decisions referenced as `D<n>` are in AGENTS.md §3.

## Goal

The prepackaged test runner of D3: a `TestRunner` (C2 §6) that runs the showcased repo's **own** Vitest in-process through the Vitest Node API, with the repo's own config, and returns the **raw** test hierarchy (file → suite → test, with status, duration and errors). It streams updates in watch mode (D18), never throws on failing tests (D8) and leaves the process as it found it (B2). Mapping results to units is the module's job (D42), never the runner's. The technique is proven by spike 1 (`01c-spike-results.md`, D94); this plan turns it into a package.

## 1. Where the code lives

```
packages/tsshowcase-testrunner-vitest/src/
  index.ts                 default export: VitestTestRunner (re-exported from its unit)
  vitest-test-runner/      exported unit: the VitestTestRunner class (README, tests; no showcase: a class, D37)
  vitest-types/            internal: minimal structural types of the Vitest Node API parts we use (§9)
  locate/                  internal: config lookup, the run plan (§3), resolving `vitest/node` and its version (§4)
  detect/                  internal: detect() (§2)
  report/                  internal: Vitest TestModule → TestFileResult mapping, errors, paths (§6)
  session/                 internal: one Vitest instance: create, start, reporter, close, process guard (§5, §7)
```

- **Runtime dependencies:** `@ofzza/tsshowcase-api` (C1), `semver`. **No `vitest` dependency of any kind** (01b K20, D94): the repo's Vitest is loaded at runtime, and the types we need are our own structural ones (§9). An ESLint `no-restricted-imports` rule bans `vitest*` imports in the package's non-test sources, because a stray `import 'vitest/node'` would resolve through this repo's hoisted Vitest and pass every test (review 28).
- `static apiVersion = '^0.1'`, `static displayName = 'Vitest'`.
- The class is constructed with `PluginHost` (C6 §8.4) and keeps only the logger from it.

## 2. `detect(repo)` (D42)

True when the repo **declares intent** to use Vitest, whether or not it is installed:

- `vitest` is in `dependencies` / `devDependencies` / `optionalDependencies` of the root manifest (`repo.info?.raw`, B7) or of any `PackageInput.info.raw`; or
- a Vitest config file (§3.1) exists at the repo root or a package directory, looked up with `repo.files.list(dir)` without `ignored: true` (D108, C4 recorded point): an ignored config doesn't exist.

Not a criterion: `vitest/node` merely resolving (a transitive, hoisted Vitest in a Jest repo would match). Detection is cheap and never imports Vitest. A root or package `vite.config.*` alone is not enough either (Vite apps without tests would match); it counts for the run plan (§3) once intent is declared.

Why "declared, not resolvable": a repo that declares Vitest but hasn't run `npm install` gets a clear "not installed" reason on its units (§4), not a silent "no runner detected".

**Jest and other tools:** `detect` is false; with no other candidate, units are `unavailable` (D74). v1 ships no Jest runner; a custom runner is a `file:` reference (D114, documented in C13).

## 3. Where Vitest runs: the run plan (K25, 01d L7; D132, refined by D134)

The runner, not the module, decides where and how many times Vitest runs, because only the runner knows Vitest's conventions (C2 amendment B7). `testRunner` is a root-only module option (D136), so one runner serves the whole repo.

### 3.1 Config lookup (review 4, 7)

Vitest's own discovery differs by major: 3.x and 4.x walk **up** from the root (past the repo root, even past `/`), 5.x looks only in the root. The runner therefore looks configs up **itself**, through `repo.files.list(dir)` (ignored = nonexistent, D108), **in the instance directory only**, and always passes `config: <absolute path>` or `config: false` to Vitest. Every major then behaves the same, and a gitignored config is never loaded.

- Config names, in Vitest's order: `vitest.config.*`, then `vite.config.*`, each with `ts`, `mts`, `cts`, `js`, `mjs`, `cjs`.
- **Vitest 3.x only:** `vitest.workspace.*` and `vitest.projects.*` (`ts`, `mts`, `cts`, `js`, `mjs`, `cjs`, `json`, `jsonc`) count as root configs and are passed through Vitest's `workspace` option. On 4.x and 5.x, which dropped workspace files, such a file is ignored with a warning "`vitest.workspace.ts` is ignored by Vitest ≥ 4; move it to `test.projects`". The major is known before the plan is final (§4).

### 3.2 The rules

- A package **qualifies** when it declares Vitest, has a Vitest or Vite config (§3.1), or the root declares Vitest (hoisted installs, review 3).
- **Root mode** (one instance at the repo root) when any of:
  - the repo is single-package (`repo.layout === 'single'`, B7), or has no packages at all;
  - the root has a `vitest.config.*` (or, on 3.x, a workspace file);
  - the root has a `vite.config.*` and no package has its own Vitest or Vite config (review 8: a root playground app's `vite.config.ts` doesn't swallow the packages' own configs).
- **Per-package mode** otherwise: one instance per qualifying package, with `root` = the package directory and the package's own config (§3.1); `vitest/node` resolves **from that package** (pnpm and nested installs, L7; hoisted installs find the root copy by walking up, as Node does). Packages that don't qualify are not run (their units get "available, zero files" from the module, C2 §4.2; `verify` already reports missing tests).
- **A root config wins:** in root mode, packages with their own config that the root's `projects` doesn't list are not run separately (Vitest's semantics; documented in the README).
- The plan is computed once per `run`/`watch` call. During `dev` it is **frozen**: a package, package config or root config added later is picked up after restarting `dev` (review 17; documented, recorded for C11).

### 3.3 Several instances

- **`run`:** instances run **sequentially** (the process guard, §7, can't be shared by concurrent runs).
- **`watch`:** instances start sequentially (each first run completes before the next starts); afterwards their reruns may overlap (review 16).
- Reports are merged: files concatenated (paths are repo-relative, so they never collide across packages); `unhandledErrors` concatenated, each message prefixed with the instance's directory (`packages/ui: …`) in per-package mode.
- **An instance that fails** (not installed, unsupported version, broken config, crash) doesn't sink the others (D135): its directory and error go into `RawTestReport.failedRoots` (B7), the module marks units under that directory `unavailable` with the error message, and a copy goes into `unhandledErrors` for diagnostics; a warning is logged. The report is `completed` while at least one instance ran; when every instance fails (or the one root instance fails), it is `status: 'error'` with the first instance's error (others in `unhandledErrors`) and `failedRoots` listing all of them.

## 4. Resolving Vitest and the version range (D94, D133)

- Per instance: `createRequire(path.join(dir, 'package.json')).resolve('vitest/node')`, then `import(pathToFileURL(entry).href)`. No `chdir`, no pool settings (spike 1). Never assume where `vite` comes from (a dependency in 3.2, dependency and peer in 4, a peer only in 5).
- Version: `vitest/package.json` resolved the same way (exported explicitly in 4 and 5, through the `./*` wildcard in 3.x), else the nearest `package.json` above the resolved entry whose `name` is `vitest`. The version is read before the plan is final, for the 3.x workspace rule (§3.1).
- **Supported range `^3.2.0 || ^4.0.0 || ^5.0.0`** (3.2 introduced `test.projects`). Below it: the instance fails with "Vitest 2.1.9 is not supported by `@ofzza/tsshowcase-testrunner-vitest` (supported: ^3.2 || ^4 || ^5)". **Above it (D133):** tried, with a one-time warning "Vitest 6.0.0 is newer than the supported range; results may be incomplete".
- Not resolvable although declared: "Vitest is declared but not installed (`vitest/node` doesn't resolve from `packages/ui`); install the repo's dependencies".

## 5. A Vitest session (spike 1)

```ts
// ≥ 5 (and newer majors): options first; 3.x/4.x: the mode-first form (deprecated in 5, review 18)
const vitest = major >= 5
  ? await createVitest({ root, config, watch, reporters: [reporter], ...OVERRIDES }, {}, { stdout, stderr })
  : await createVitest('test', { root, config, watch, reporters: [reporter], ...OVERRIDES }, {}, { stdout, stderr });
await vitest.start(filters);   // run: resolves after the run; watch: after the first run
await vitest.close();          // run: always (finally); watch: on signal abort
```

- **Overrides** (inline options win over the user's config): `reporters: [ours]` (no default reporter; nothing printed), `coverage: { enabled: false }`, `ui: false`, `open: false`, `api: false` (no ports), `passWithNoTests: true` (keeps Vitest from setting `exitCode = 1` when nothing matched; it doesn't stop `start()` from throwing, review 31). Everything else is the user's config: pools, environments, `typecheck` (type tests appear when the user enables them, D38), setup files, browser mode (supported as configured, not covered by our tests; documented).
- **Environment (D137):** the runner sets no variables of its own. Unlike the `vitest` command, a Node API session doesn't set `TEST`, `VITEST` or `NODE_ENV ??= 'test'` in the main process, and Vite then sets `NODE_ENV=development`; workers inherit that `NODE_ENV` (they get `TEST`/`VITEST` from the pool regardless). So under tsshowcase, test files see `NODE_ENV=development` unless the user set it, and config files that branch on `process.env.VITEST` take their non-Vitest branch. Documented in the package README; everything is restored afterwards (§7).
- **Output:** `stdout`/`stderr` are in-memory sinks forwarded line by line to the host logger at debug level (Vitest's own warnings, e.g. "Testing types with tsc and vue-tsc is an experimental feature", end up there). Config-load errors that Vitest prints with `console.error` (spike problem 4) are captured while `createVitest` runs, **still forwarded** to the original `console.error` (other `dev` output isn't swallowed, review 25), and become the instance error's message.
- **Errors:** `createVitest` throwing (broken config) → the instance fails (§3.3) with the captured message; `start()` throwing `VITEST_FILES_NOT_FOUND` (code stable across 3–5; thrown in watch mode too, review 13) → `completed` with `files: []`, and in `watch` the instance **stays alive** to pick up new test files; any other crash inside `start` → the instance fails, `close()` still runs.
- **Filters (B7, review 6):** `TestRunOptions.files` are repo-relative file or directory paths. Vitest matches relative filters against each project's directory and resolves them against `process.cwd()`, so the runner passes them as **absolute** paths (directories with a trailing `/`), which Vitest matches by prefix in every project and instance.
- **Abort:** `signal` aborted before or during a run → `close()`, and the call **resolves** with `status: 'error'`, `error.message: 'aborted'` (never throws, B7). In `watch`, aborting after the first run just stops watching and resolves nothing further.
- **One run at a time per runner instance:** calls to `run`/`watch` on one `VitestTestRunner` are queued; a `watch` holds the queue until it is aborted. A queued call whose `signal` aborts leaves the queue at once and resolves `error`/`aborted` (review 26). (C6 caches instances per invocation, so this covers a module calling twice.)

## 6. Mapping the report (C2 §6)

| Vitest (reporter API)                                             | Raw report                                                                                                     |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `TestModule.moduleId` (absolute)                                  | `file`: relative to `repo.root`, POSIX separators; tried against both `repo.root` and its real path (symlinked roots, macOS `/var` → `/private/var`, review 23); a file outside both is dropped with a debug log |
| `TestModule.project.name`                                         | `project`; empty name → omitted                                                                                |
| `TestModule.state()`                                              | `queued` → `queued`, `pending` → `running`, `passed`/`failed`/`skipped` as is                                   |
| `TestModule.diagnostic().duration`                                | `durationMs`                                                                                                   |
| `TestModule.errors()` (a syntax error: no children; an `afterAll` failure: with children) | `error` (the first; the rest appended to its message)                                   |
| `TestSuite` (`name`, `state()`, `errors()`, `children`)           | `{ type: 'suite', …, error? }`, recursively (suite errors, e.g. a failing `beforeAll`, B7, review 20)          |
| `TestCase` (`name`, `result().state`, `options.mode`, `diagnostic()`) | `{ type: 'test', … }`: `options.mode === 'todo'` → `todo`; `pending` → `running`; `duration` → `durationMs`; first error → `error` |
| `onTestRunEnd(_, unhandledErrors)`                                | `unhandledErrors` (spike problem 7: TS syntax errors that make type tests pass silently; the nested spike saw TS errors in non-test files land only here) |

- `children` is a `TestCollection`: iterated in Vitest's order (source order).
- **Final reports** (the `run` result and every `onTestModuleEnd`): a test or suite still `pending`/`queued` (after a crash or an abort) maps to `skipped`, never `running` (review 21).
- **`TestError`:** `message`, `stack`, `expected`, `actual` from Vitest's serialised error, ANSI escapes stripped (`util.stripVTControlCharacters`). Absolute paths inside messages and stacks are left as is: the CLI relativises free text (L2, C11). C7 guarantees only the `file` fields.
- Results for ignored files are **not** filtered: Vitest decides what runs; such files map to no unit and end up in the module's `unmapped` (as do tests in out-of-scope directories, C4 M6).

## 7. Process guard (B2, B7; spike problems 1–2; review 1, 24)

- **`process.env`:** snapshotted (a shallow copy) before the first instance is created and restored (added keys deleted, changed ones reset) **after the last `close()`**: after every `run`, and when watching stops. Restoring it mid-watch would change the `NODE_ENV` that workers spawned for later reruns inherit. Safe because `dev` never runs a production `vite build` in the same process; C15's `build` sets `mode`/`NODE_ENV` explicitly anyway (D94), and the C15 dev server must not rely on `process.env.NODE_ENV` (recorded).
- **`process.exitCode`:** restored after every `run`; in `watch`, after every `onTestRunEnd` (Vitest sets it before that hook), and **only when Vitest changed it**: compared with the value captured at `onTestRunStart`, so the CLI's own value is never clobbered.
- **Process listeners (review 1):** every Vitest instance registers `SIGINT`, `SIGTERM` and `exit` handlers that call `process.exit()` within 1 ms, plus an `unhandledRejection` handler that prints and exits, until `close()`. In `dev`, Ctrl-C would kill the CLI before `DevSession.close()` and `dispose()` run, and any unrelated unhandled rejection would kill the process. The guard diffs `process.listeners(event)` for those four events around `createVitest` and removes the ones Vitest added; the CLI's own shutdown path calls `close()` (C11). Many per-package instances therefore also never trip `MaxListenersExceededWarning`.

## 8. Watch mode and streaming (D18)

`watch(options, onUpdate)` creates the instances with `watch: true` and resolves with the merged report of the first full run. After that, per instance:

- `onTestRunStart(specifications)` → `onUpdate({ files: [each file about to run, status running] })`, its previous children kept with `queued` status, so the page shows "running" without emptying the file (review 30: one hook, fired once per rerun);
- `onTestModuleEnd` → `onUpdate({ files: [final file] })` (full replacement per `(file, project)`, B2);
- `onTestRunEnd(_, unhandledErrors)` → `onUpdate({ files: [], unhandledErrors })`, then the exit-code restore (§7);
- `onTestRemoved(id)` (a public reporter hook in 3.2, 4 and 5, fired by Vitest's watcher on unlink, review 12) → `onUpdate({ removed: [every cached (file, project) for that path] })`. Still best-effort: the CLI's dev watcher stays the reliable source (C2 §5.7, B2).
- **Config changes (review 2):** Vitest re-creates its server on a config edit but only its CLI re-runs tests; the runner registers `vitest.onAfterSetServer(() => vitest.start(filters))`, with the same error handling as the first `start` (§5: no files → an empty files update; any other error → an `unhandledErrors` update).
- Updates fired within one event-loop turn are coalesced into one `RawTestUpdate`.
- **`unhandledErrors` in updates** are the merged set across instances and **replace the previous set** (an empty array clears it); an update without the field leaves it unchanged (B7).
- New test files are Vitest's own watcher's business.

## 9. Structural types (`vitest-types/`)

`TestModule`, `TestSuite`, `TestCase`, `TestCollection`, the reporter hooks of §8, both `createVitest` signatures (§5) and the `Vitest` members we call (`start`, `close`, `onAfterSetServer`), typed as what the three supported majors share. **Type test (review 28):** a `*.test-d.ts` checks our structural types against this repo's own (root devDependency) Vitest types, without adding a package dependency; the fixture matrix (§10) guards the other majors at runtime.

## 10. Fixtures and tests

**Fixtures** (self-contained, committed lockfiles, installed by `fixtures:install`, C1 §5):

| Fixture                   | Purpose                                                                                                                                                                                                                 |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `vitest-basic`            | Single package on **Vitest 5**: passing, failing (with `expected`/`actual`), skipped and `todo` tests, nested `describe`s, a failing `beforeAll` in a suite, a file with a syntax error, `typecheck` enabled with a `*.test-d.ts` (incl. a failing type test) |
| `vitest-v3`, `vitest-v4`  | The same small suite on Vitest 3.2.x and 4.x (the D94 range; one source per fixture, kept in sync by hand); `vitest-v3` also has a `vitest.workspace.ts`                                                                 |
| `vitest-projects`         | npm workspace with a root `vitest.config.ts` using `test.projects` (two packages, one file matched by two projects)                                                                                                      |
| `vitest-per-package`      | npm workspace with **`.npmrc` `install-strategy=nested`** (proven by spike: the committed lockfile records the nested paths, `npm ci` reproduces them): no root config and no root Vitest; package `a` on Vitest 5, package `b` on Vitest 3.2 (each resolves its own copy), package `c` without Vitest |
| `vitest-hoisted`          | npm workspace, Vitest only in the root `devDependencies`, no root config: one package with a `vite.config.ts` `test` block, one with no config (review 3)                                                                 |

Generated at test time with `copyFixture(…, { linkNodeModules: true, files })` instead of more lockfiles: a broken `vitest.config.ts` (throws), no test files, Vitest declared but not installed (no `linkNodeModules`), **an unsupported version** (a fake `node_modules/vitest/package.json` with `2.1.9` next to a resolvable entry stub; the version check never imports it, review 27), a fake newer major (warned once), a Jest-only manifest (`detect` false), a gitignored `vitest.config.ts` (not detected, not loaded), a root playground `vite.config.ts` next to package configs (per-package mode), a `vitest.workspace.ts` on Vitest 5 (ignored, warned).

**Tests run in-process (nested spike).** Our tests run inside Vitest workers, and creating the fixture's Vitest from there works on 3.2, 4 and 5, under the `forks` and `threads` pools, with `--no-isolate`, with typecheck and watch, and for fixtures sharing this repo's own Vitest install; the outer run's results, reporter output and exit code stay clean. The only difference is that inner test files inherit the outer worker's `NODE_ENV=test` instead of `development` (D137: we don't touch env), which no test depends on. So the integration tests call the runner directly; no child-process harness. Integration tests carry a raised timeout (30 s; a run takes about 1 s per instance).

**Tests:**

- `report/` (unit, with fake Vitest objects): every row of §6, nested suites and suite errors, file errors with and without children, todo vs skipped, running/queued in updates vs `skipped` in final reports, ANSI stripping, symlinked roots, paths outside the repo dropped, Windows separators normalised.
- `locate/` (unit + fixtures): config lookup in the instance directory only (a parent config is never used, on 3.x and 4.x too), ignored configs, every rule of §3.2, 3.x workspace files and the ≥ 4 warning, resolution from the package, version read, the range incl. `3.1.x` rejected and a newer major warned once.
- `detect/`: each criterion, the generated cases above, ignored config files not counted.
- `session/` (every Vitest fixture): `run` end to end on Vitest 3, 4 and 5 (same expected report apart from durations and error message/stack text, which differ by major); projects (the doubled file has two `(file, project)` entries); per-package merge, with `vitest/node` asserted **not** to resolve from the `vitest-per-package` root and each package reporting its own Vitest version; hoisted (both packages run); partial failure (`failedRoots`, `unhandledErrors` copy, warning) and total failure (`error`); unhandled errors (the TS syntax error case); no tests → `completed`, `files: []`; broken config → `error` with the captured message, also forwarded to `console.error`; not installed; unsupported; `files` filters as repo-relative files and directories, across projects and per-package instances, independent of `process.cwd()`; abort before and during a run and while queued; **`process.env` and `process.exitCode` identical before and after a failing run**; **no Vitest listeners left on `SIGINT`/`SIGTERM`/`exit`/`unhandledRejection`** during and after a run; queueing of concurrent calls.
- `watch`: the first report; editing a test file emits `running` then the final file, then a run-end update with `unhandledErrors`; deleting it emits `removed`; editing the config re-runs; starting on an empty fixture and adding a test file emits an update; `exitCode` restored after a failing rerun only when Vitest changed it; an unrelated unhandled rejection and a `SIGINT` listener check while watching (no Vitest exit handlers installed); on `vitest-per-package`, an edit in one package updates only that package's files; env restored and no hanging handles after abort.
- **CLI built-in name (C6 L14):** in the CLI package, `loadTestRunner('vitest')` loads this package through C6's built-in table, and auto-detection over `vitest-basic` returns it as the only match.
- `vitest-test-runner/` type test: `VitestTestRunner satisfies TestRunnerConstructor`; `vitest-types/` type test against this repo's Vitest (§9).

## 11. Documentation

- **Package README** (one place per fact, D56): what it does, the supported Vitest range and the newer-major warning, detection rules, the run plan (root vs per-package mode, config lookup only in the run directory, "a root config wins", a root `vite.config` only counts without package configs, 3.x workspace files), the plan is fixed for a `dev` session, what we override in the user's config (§5, incl. what `passWithNoTests` does), the environment tests see (`NODE_ENV=development` unless set, `VITEST` unset in config files, D137), what happens when one package's Vitest fails (units `unavailable`), browser mode untested, Jest not supported (write a custom runner, C13), the README sample `testRunner: 'vitest'` (a root-only option, D136).
- `vitest-test-runner/` unit README: the class, its constructor and methods.
- `fixtures/README.md`: the new fixtures, why `vitest-per-package` uses `install-strategy=nested` (keep the `.npmrc` so regenerating the lockfile stays nested), and that `copyFixture` links every `node_modules` (C4 amendment).
- C13 picks up: writing a custom runner (Jest example), browser mode.

## Acceptance criteria

1. On `vitest-basic`, `vitest-v3` and `vitest-v4`, `run` returns the documented raw report, identical across the three majors apart from durations and error message/stack text.
2. `vitest-projects`, `vitest-per-package` and `vitest-hoisted` produce merged reports with correct `project` values and repo-relative paths; per-package mode resolves each package's own Vitest (5 and 3.2 side by side).
3. `process.env` and `process.exitCode` are unchanged after a failing `run`, and `exitCode` after a failing watch rerun; no Vitest process listeners remain installed.
4. `watch` streams `running`, final, run-end and `removed` updates, re-runs after a config edit, and stops cleanly on abort (no hanging handles).
5. Broken config, not installed, unsupported version, a failing package among working ones, no tests and abort each produce the documented outcome without throwing.
6. `loadTestRunner('vitest')` and auto-detection load this package through the CLI (C6 L14).
7. The package has no `vitest` dependency of any kind and its sources import no `vitest*` module; `npm run ci` passes.

## Out of scope

Mapping results to units, `unmapped`, `TestResults` (the module: C10 for `-base-react`); defaulting absent units, relativising error text (L2), caching results, strict mode (C11); surfacing `unhandledErrors` and `unmapped` in the data (L3, deferred); a Jest runner; Vitest browser-mode coverage in our tests; mixing runners per package (D136).

## Recorded for later chunks

- **C8:** `-base-react`'s `testRunner` option is root-only: its schema description says so (D136).
- **C10:** `runTests` passes `{ repo: ctx.repo, files?, signal }` (B7) and never picks a root; maps `(file, project)` results to units; marks units under `failedRoots` directories `unavailable` with the error message (D135); counts `queued` tests as `running` in `TestSummary` (review 22); suite `error`s render with the suite; a runner instance serialises its calls (§5), so a module that wants run + watch uses one `watch`.
- **C11:** relativise `repoRoot` in `TestError` strings (L2); `failedRoots` errors and their `unhandledErrors` copies are part of L3; `dev` never runs a production build in the process hosting the watcher, and the C15 dev server doesn't rely on `process.env.NODE_ENV` (§7); the CLI's own `SIGINT`/`SIGTERM` handling aborts `runTests` and then disposes (Vitest's exit handlers are removed, §7); the run plan is frozen per `dev` session (a new package or Vitest config needs a restart, §3.2).
- **C13:** writing a custom runner (e.g. Jest) as a `file:` plugin; Vitest browser mode is untested; "a root Vitest config wins over package configs"; the test environment under tsshowcase (D137).
- **C14:** the fixture matrix installs Vitest 3, 4 and 5; keep Dependabot ignoring fixture majors (C1).
- **C15:** the dev server doesn't rely on `process.env.NODE_ENV` (§7).

## C2 amendment B7 (re-approved with this plan, applied)

- **§6 `RunnerRepoInput`:** gains `layout` and `info?` (root manifest), so `RepoInput` stays assignable and C6/C10 pass `ctx.repo` as is (review 19):

  ```ts
  export interface RunnerRepoInput {
    readonly root: string; // absolute
    readonly layout: 'single' | 'workspace'; // B7
    readonly info?: PackageInfo; // B7: the root manifest, if any
    readonly packages: readonly PackageInput[];
    readonly files: RepoFiles;
  }
  ```

- **§6 `TestRunOptions`:** `root` and `repoRoot` are replaced by `repo: RunnerRepoInput`; the runner decides where and how often its tool runs (D132) and reports paths relative to `repo.root`. `files` become repo-relative file or directory paths (review 6):

  ```ts
  export interface TestRunOptions {
    readonly repo: RunnerRepoInput; // B7: replaces root/repoRoot; the runner decides where its tool runs
    readonly files?: readonly RepoPath[]; // B7: optional filter, repo-relative files or directories (prefix match)
    readonly signal: AbortSignal;
  }
  ```

- **§6 `RawTestReport.failedRoots?: readonly { readonly dir: RepoPath; readonly error: TestError }[]`** (D135): directories where the tool couldn't run while others did; the module marks units under them `unavailable` with the error's message. With `status: 'error'` it lists every failed directory.
- **§6 `TestNode` suite:** gains `error?: TestError` (e.g. a failing `beforeAll`); the file-level `error` comment becomes "a file-level failure (e.g. a syntax error, then without children, or an `afterAll` failure)" (review 20).
- **§6 `RawTestUpdate.unhandledErrors`:** when present it replaces the previous set (an empty array clears it); absent leaves it unchanged.
- **§6 rules:** an aborted `run`/`watch` resolves with `status: 'error'` rather than throwing. The process rule is refined (review 10): in `run`, `process.env` and `process.exitCode` are restored after the run; in `watch`, `exitCode` after every rerun (only when the tool changed it) and `env` when watching stops; runners also remove process-wide signal and exit handlers their tool installs (review 1).
- **§4.2:** a unit under a `failedRoots` directory is `unavailable` with that error's message (D135), not "available with zero files".
- **§5.7:** the sample passes `{ repo: ctx.repo, signal: ctx.signal }`; "even when `TestRunOptions.root` is a package directory" becomes "whichever directories the runner ran its tool in"; `testRunner` is read from the root options only (D136).
- **§5.11:** the `testRunner` convention says "a root-only option" (D136).
- **§12 / README:** the minimal runner sample and the compat-guard fixture follow the new `TestRunOptions`.

## Other amendments (approved with this plan, applied)

- **C4 §11.1 (`copyFixture`, nested-install spike):** `linkNodeModules` symlinks **every** `node_modules` directory of the fixture that isn't itself inside a `node_modules` (the root and each package that has one), at the same relative path in the copy; linking only the root one breaks nested installs. Documented caveat: the root `node_modules` workspace links are relative, so through the symlink they point at the original fixture's packages; a test that edits package sources in the copy and imports them by package name needs a real root `node_modules` with fresh links (added when a test needs it). Resolved paths are the original's real paths. Own test: a nested-install fixture copy resolves like the original.
- **C6 §7 / §9.2 (D136):** a `testRunner` key in a package config's `moduleOptions` is warned about ("`testRunner` is read from the root config only") and dropped before validation. Since module option names are the module's business, the CLI warns only for the conventional `testRunner` key (C2 §5.11).
- **C6 §11:** `detect` and the runner receive `RunnerRepoInput` built as `ctx.repo` itself (B7).

## Super-plan R8 and AGENTS edits (applied)

- **R8 (C7 planning, D132–D137, review `07a-c7-plan-review.md`):**
  - C7: "Runs the showcased repo's Vitest … using the repo's own vitest install and config" gains "the runner decides where it runs: root or per package (D132, D134)"; `detect` becomes "Vitest declared in a manifest, or a `vitest.config.*`/workspace file" (no `vite.config` `test` block; not "`vitest/node` resolves"); "`vitest/node` resolved from the repo root" → "from the directory each instance runs in"; "restore `process.env` and `process.exitCode` after each run" → the refined rule (B7), plus removing Vitest's process handlers; newer majors tried with a warning (D133).
  - C10: `runTests` passes `ctx.repo`; `failedRoots` → `unavailable` (D135).
  - C2 bullet: amendment B7.
- **AGENTS:** D132–D137 in the log (D132, D133 already recorded; D134–D137 recorded at triage); D94 status "refined by D132–D134"; D69 note: `testRunner` is root-only (D136); open points: C7 row removed; C8 row gains "`testRunner` root-only in the schema (D136)"; C10 row gains "`ctx.repo` to the runner, `failedRoots` → `unavailable`, `queued` counted as running, suite errors"; C11 row gains "the CLI's signal handling aborts `runTests` (Vitest's exit handlers removed), the run plan frozen per `dev` session, `failedRoots` in L3"; C13 row gains "custom runners (Jest), browser mode, the test environment under tsshowcase (D137)"; C15 row gains "the dev server doesn't rely on `process.env.NODE_ENV`"; C6 note for the `testRunner` package-config warning; the C2, C4 and C6 tracker rows note the amendments; C7 marked approved in the tracker.

## Open questions

- ✅ **Q1 Where Vitest runs (K25, L7; D132):** the runner decides (§3); C2 amendment B7: `TestRunOptions.repo` replaces `root`/`repoRoot`.
- ✅ **Q2 Vitest majors above the range (D133):** tried with a one-time warning; below 3.2 refused (§4).
- ✅ **Q3 Run plan refinement (review 3, 4, 7, 8; D134):** the runner looks configs up itself, only in the instance directory, and passes `config` explicitly; packages qualify by declaring Vitest, having a Vitest/Vite config, or a root Vitest declaration; root mode for single packages, a root `vitest.config` (or 3.x workspace file), or a root `vite.config` without package configs (§3).
- ✅ **Q4 A failing instance among working ones (review 9; D135):** `failedRoots` in the report; its units are `unavailable` with the reason (B7).
- ✅ **Q5 Per-package `testRunner` (review 11; D136):** root-only; a package config setting it is warned about and dropped (C6).
- ✅ **Q6 The test environment (review 5, nested spike; D137):** left as is: the runner sets no variables (tests see `NODE_ENV=development` unless set; `VITEST` unset in config files); it only restores what Vitest changed. Claude recommended mirroring the `vitest` command's `TEST`/`VITEST`/`NODE_ENV ??= 'test'`; the user chose to leave the environment alone.
