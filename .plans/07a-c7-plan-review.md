# C7 plan review (fresh eyes) and spikes, triaged

A clean-context agent reviewed all plans, with a focus on the C7 draft (round 2) and the changes it brings to other plans (2026-10-19). It checked its findings against the Vitest 3.2.7, 4.1.12 and 5.0.5 sources ([V] = verified there, [I] = inferred). Two spikes ran alongside, in Claude's scratchpad only (no sudo, no global installs). Every finding is applied in C7 round 3 unless noted; decisions are D134–D137.

## Spikes

| Spike | Result | Applied |
| ----- | ------ | ------- |
| **Nested Vitest:** the runner inside this repo's own Vitest workers (outer Vitest 5, typecheck on; inner 3.2.7, 4.1.12, 5.0.5; Node 24.15) | Works in-process as is: correct results under `forks` and `threads`, with `--no-isolate`, with inner typecheck and watch, and for a fixture sharing the outer Vitest install; the outer run's output, results and exit code stay clean; no hanging handles; about 1 s per inner run. Only difference: inner test files inherit `NODE_ENV=test` (no result changed). A child-process harness also works (about 50 ms overhead) but isn't needed. Side findings: TS errors in non-test files land only in `unhandledErrors` (supports L3); Vitest writes warnings to the stdout stream it's given | C7 §10: tests call the runner in-process, no harness; §5 output forwarding |
| **Nested npm installs:** a workspace where `vitest/node` resolves only per package (npm 11.12.1) | `install-strategy=nested` in the root `.npmrc` works: the root `node_modules` holds only workspace links; package `a` resolves Vitest 5.0.5, `b` 3.2.7, root and `c` nothing. The lockfile itself records the nested paths, so `npm ci` reproduces the layout (about 1 s). `copyFixture`'s root-only `node_modules` link breaks it; every package `node_modules` must be linked. Caveats: resolved paths are the original's real paths; root workspace links point at the original's packages | C7 §10 `vitest-per-package`; C4 §11.1 amendment |

## Findings

| # | Sev | Finding | Resolution |
| - | --- | ------- | ---------- |
| 1 | high | [V] Each Vitest instance registers `SIGINT`/`SIGTERM`/`exit` handlers that `process.exit()` within 1 ms, and an exiting `unhandledRejection` handler, until `close()`: Ctrl-C in `dev` bypasses abort → `close` → `dispose`; > 10 instances warn | §7: the guard removes the listeners Vitest added; tests; B7 rule; recorded for C11 |
| 2 | high | [V] After a config edit in watch mode Vitest re-creates its server but only its CLI re-runs tests | §8: `onAfterSetServer(() => start(filters))` with error handling; test |
| 3 | high | The run plan runs nothing for a hoisted monorepo (Vitest only at the root, no root config, packages with a `vite.config` `test` block or no config) | **D134**: a package qualifies by a root Vitest declaration too, and by a `vite.config`; new fixture `vitest-hoisted` |
| 4 | high | [V] Config discovery differs: 3.x/4.x walk up past the root, 5.x doesn't; a gitignored config still loads | **D134**: the runner looks configs up itself in the instance directory only and always passes `config` |
| 5 | high | [V] The Node API doesn't set `TEST`/`VITEST`/`NODE_ENV ??= 'test'` like the `vitest` command: tests see `NODE_ENV=development`, configs branching on `process.env.VITEST` take the wrong branch; §7's rationale about workers was wrong (workers get `TEST`/`VITEST` from the pool; only `NODE_ENV` is inherited) | **D137** (the user's choice, against Claude's recommendation to mirror the CLI): environment left as is, documented; §7 rationale corrected |
| 6 | medium | [V] Substring filters are matched per project directory and resolved against `process.cwd()` | B7: `files` are repo-relative paths; the runner passes absolute paths |
| 7 | medium | [V] `vitest.workspace.*` exists only in 3.x (with `vitest.projects.*`) | §3.1: counted on 3.x only, warned on ≥ 4 |
| 8 | medium | A root `vite.config.*` (e.g. a playground) forces root mode and swallows package configs | **D134**: counts only when no package has its own config |
| 9 | medium | A package whose instance failed shows "0 tests", against D42 | **D135**: `RawTestReport.failedRoots`, units `unavailable` (B7, C10) |
| 10 | medium | B7 missed the refined restore rule (C2 §6 says "after every run") | B7 rule; super-plan R8 |
| 11 | medium | Per-package `moduleOptions.testRunner` (D69) conflicts with a repo-wide runner | **D136**: root-only; C6 warns and drops it in package configs; C8 schema text |
| 12 | medium | [V] `onTestRemoved` is a public reporter hook in 3.2–5 | §8 emits `removed`; still best-effort |
| 13 | medium | [V] `VITEST_FILES_NOT_FOUND` is thrown in watch mode too | §5: empty first report, instance kept alive; test |
| 14 | medium | Plan drift not recorded (super-plan C7 bullets, D94 status, C2 status, AGENTS rows) | "Super-plan R8 and AGENTS edits" section |
| 15 | medium | `copyFixture` links only the root `node_modules`; nested install unverified | Spike verified nested installs; C4 §11.1 amendment; test asserts the root doesn't resolve Vitest |
| 16 | medium | Several live instances unproven; start order unspecified | §3.3: sequential first runs, overlapping reruns; per-package watch test |
| 17 | medium | The run plan is frozen during `dev` | §3.2 documented; recorded for C11 |
| 18 | medium | [V] Vitest 5 deprecates the mode-first `createVitest` | §5: options-first on ≥ 5 |
| 19 | low | "Single-package" undefined for the runner; `packages: []` | B7: `RunnerRepoInput.layout`, `info?`; no packages → root mode |
| 20 | low | [V] Suite errors (`beforeAll`) lost; file errors can have children | B7: suite `error?`; §6 wording |
| 21 | low | [V] Leftover `pending` in final reports would read as running | §6: `skipped` in final reports |
| 22 | low | `queued` children have no summary counter | Recorded for C10 (count as running) |
| 23 | low | Symlinked roots drop every result | §6: relativise against the real path too |
| 24 | low | Restoring `exitCode` in watch can clobber the CLI's own value | §7: only when Vitest changed it |
| 25 | low | Capturing `console.error` swallows concurrent `dev` output | §5: captured and forwarded |
| 26 | low | Aborting a queued call unspecified | §5 |
| 27 | low | Harness `jiti` dependency; Vitest 2.1 fixture adds an old major; "identical across majors" too strict | Harness dropped (spike); fake `package.json` instead of a 2.1 install; acceptance excludes message/stack text |
| 28 | low | [V/I] Structural types can be type-checked against the repo's own Vitest; a stray `vitest/node` import would pass via hoisting | §9 type test; §1 `no-restricted-imports` |
| 29 | editorial | `repoRoot` → `repo.root`; Vitest 3.x exports `package.json` via `./*`; merged `unhandledErrors` in updates | Applied |
| 30 | editorial | [V] `onTestRunStart(specifications)` is the single point for "running" | §8 |
| 31 | editorial | [V] `passWithNoTests` only prevents `exitCode = 1`, not the throw | §5 wording; README |

Verified as correct by the reviewer: `vitest/node` resolves via `createRequire` in all three majors; `VITEST_FILES_NOT_FOUND` is stable; the reporter hooks used exist in all three; the `TestModule`/`TestCase` members used exist (`children` is a `TestCollection`); `start()` resolves after the first run in watch mode; `exitCode` is set before `onTestRunEnd`.
