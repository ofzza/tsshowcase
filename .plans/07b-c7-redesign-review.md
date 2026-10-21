# C7 redesign amendment: choices and review (plan 8)

> Plan 8 of the representation redesign's application (`08b-representation-redesign.md` §12, C7; `08e` E7). Edits **Chunk C07** (status line, rewritten sections, new §5.1, "Redesign amendment"), wording in **Chunk C06** (§11, §14) and **Chunk C02** (§4.4) for D209, and **AGENTS** (progress note, tracker row, D209, open points). Status: **drafted 2026-10-21; choices C1–C6 settled; reviewed by two zero-context agents; questions QA–QD settled and Fix 1–17 applied; **approved by the user 2026-10-21**.** D209 decided (C6).

## Checklist sources

08b §12 (C7); `02b-c2-b9-review.md` "Saved for the later plans" (C7); `04b-c4-redesign-review.md` (B: Vitest's own config names, already worded in §3.1); `05b` (nothing for C7); `06b-c6-redesign-review.md` "For later plans" (R2#25, the stale C6 lines); C2's, C4's and C6's "Recorded for later chunks" (C7); the AGENTS open points (C11: re-listing while a `watch` holds the queue).

## Choices (discussed one by one with the user)

- **C1 (1/6, ok, option a; revised by QA):** a `listTestFiles` call while a `watch` is active doesn't queue: it lists through the watch's live instances (`globTestSpecifications()` re-globs), so new and deleted test files show up, the plan stays frozen and no second Vitest is created; instances that failed at the watch's start stay in `failedRoots` until `dev` restarts. Rejected: (b) fresh instances next to the watch (a second Vitest, the guard snapshotting a changed env, a possibly different plan); (c) the CLI deriving new test files itself (Vitest conventions in the CLI, misses unrun files). C7 §3.2, §5, §5.1; recorded for C11.
- **C2 (2/6, ok):** listing outcomes mirror `run`: instances sequential under the process guard; no files → `completed`, `files: []`; a failing instance among working ones → `failedRoots`; all failing → `error` with the first error and every `failedRoots`; abort → `error`/`'aborted'`; warnings and captured config output as in §5. Rejected: any instance failure as a total `error` (one broken package would hide every missing-tests issue). C7 §5.1.
- **C3 (3/6, ok):** in `build`, listing and running are separate Vitest sessions (the listing closes its instances and restores the process state). Rejected: keeping the listing's instances for the run (Vitest's env and handlers live during the module's analysis and build; a hand-off API the contract lacks). Caching may come later as an internal optimisation.
- **C4 (4/6, ok, option a):** the `files` filter stays implemented and tested although the CLI passes none in v1 (the contract declares it; later uses such as rerunning changed files). Rejected: (b) removing it from the contract; (c) "may ignore".
- **C5 (5/6, ok):** the cwd-independence test moves `process.cwd()` outside the fixture; it covers every fixture on Vitest 5 and, on 3.x/4.x, configs without relative inline project `root`s (our fixtures avoid them); the README documents the caveat (D199); no test pins the limitation (Vitest's behaviour, not ours).
- **C6 (6/6, ok, option b → D209):** runner ids are **slugified display names**: lower case, runs of characters outside `[a-z0-9]` become `-`, trimmed, empty → `runner`, then de-duplicated (`vitest`, `vitest-2`); `RunnerTests.name` keeps the display name (`Vitest`). Resolves the clash between C7's `displayName = 'Vitest'` and C6 §11's example `'vitest'`. Rejected: (a) ids as raw display names (capitals and spaces in JSON keys); (c) a lower-case `displayName` (fixes only our runner). Wording in C6 §11, §14 and C2 §4.4.

## Review

Two zero-context reviewers, read-only: **R1 fidelity and completeness** (C7 vs 08b, D160–D209, C2 §4.4/§5.7/§6 and the checklist sources) and **R2 fit, implementer's read and Vitest assumptions** (C7 vs C1 §8, C4, C6, C15, the super-plan and the AGENTS rows; Vitest 3.2.4, 4.1.0 and 5.0.5 sources found locally). 34 findings, triaged into four questions (discussed one by one), 17 fixes (approved together), later-plan items and skips. Claude re-checked R2's high-severity finding in the 5.0.5 source (`index.1TqnEahn.js:6605`, `cli-api.De07GUeT.js:12810`) before triage.

### Questions

- **QA (1/4, ok, option a′; revises C1):** `globTestSpecifications()` does **not** re-glob: each project caches its file list after the first glob, updated only by Vitest's own watcher (never for type-test files); `clearSpecificationsCache()` resets it on 4.x/5.x only and never the type-test list. A listing during a `watch` therefore creates **fresh, short-lived instances for the watch's frozen plan** (no queueing, its own `signal`, the guard restoring the env as the watch left it); the watch's instances are never touched. Gap documented (README, C13): Vitest's watcher doesn't run a type-test file created during `dev` until restart. Dissolves R2#8, R2#12–15. Rejected: (b) `clearSpecificationsCache()` on the live instances (mutates watch state, misses type tests, 3.2 race); (c) run-reported files counted as listed in `dev` (changes D172). C7 §3.2, §5, §5.1, §7, §8, §10, §11.
- **QB (2/4, ok, option a):** no `unhandledErrors` copies of failed instances' errors (D135's "for diagnostics" copy): `failedRoots` alone carries them, so one failed package doesn't fail both `--strict=unavailable` and `--strict=unhandled` (D164), and listings and runs report failures alike. Rejected: (b) keep the copies. C7 §3.3, §5.1, §10. (R2#5, R1 3.1.)
- **QC (3/4, ok, option a):** the CLI never overlaps process-state work: runners are called one at a time; test listing and `analyze` (and directory-config loading) run one after the other, never in parallel; in `build`, `run` completes before the module's `build` (C15 sets `NODE_ENV=production` around `vite.build`). The `dev` `watch` is the accepted exception. Wording in C2 (§5.3/§5.4 comments, §5.9, a §6 rule), 08b §3.1 and D169's note ("independently of" instead of "in parallel with"); recorded for C9, C11, C15. Rejected: (b) a contract rule that `analyze` must not touch process state. (R2#6, R2#7.)
- **QD (4/4, ok, option a):** a file matched by both a project's `include` and `typecheck.include` yields two specifications (and likely two run modules) with the same `(file, project)`; the runner keeps one listing entry and merges run results into one `TestFileResult` (children concatenated, runtime first; status precedence `failed` > `running` > `queued` > `passed` > `skipped` > `todo`; durations summed; errors as §6), also in `watch` updates; verified at implementation with a generated fixture. Rejected: (b) a synthetic project name; (c) report as is. C7 §5.1, §6, §10. (R2#2.)

### Fixes (approved together)

- **Fix 1 (R1 1.1):** in-source tests (`includeSource`) listed; fixture variant and test.
- **Fix 2 (R1 1.2):** D160–D209 in the status line; the CLI built-in test asserts the id `vitest`.
- **Fix 3 (R1 1.4, R2#18):** the "nothing runs" test with a test **and** a `globalSetup` writing markers; "Unchanged" list corrected.
- **Fix 4 (R1 3.1):** §3.3 per QB.
- **Fix 5 (R1 3.2):** the CLI's shutdown aborts the runner's `signal`; the runner calls `close()`.
- **Fix 6 (R1 4.1):** open questions Q4, Q5 annotated as replaced/void by D172.
- **Fix 7 (R1 4.2):** historical banner on "Super-plan R8 and AGENTS edits".
- **Fix 8 (R1 4.3):** stale citations (D42, D135, D38, C2 §5.7) updated.
- **Fix 9 (R1 5.1):** the B7 banner notes `RunnerPackageInput[]` and listing in the process rule.
- **Fix 10 (R1 6.1):** the cwd test under the `forks` pool, cwd restored in `finally`.
- **Fix 11 (R1 6.2):** `detect` never throws; an unreadable directory counts as no config; test.
- **Fix 12 (R2#11):** §3.2 notes skipped packages run by a root `projects` config come back `outOfScope`.
- **Fix 13 (R2#16):** listings sorted by `(file, project)`.
- **Fix 14 (R2#17):** one-time warnings once per runner instance, across listing, run and watch.
- **Fix 15 (R2#4):** `changed: false` override, `related` never passed.
- **Fix 16 (R2#6, QC):** C2 §6 rule and "independently of" wording in C2, 08b, D169.
- **Fix 17 (R1 1.3, R2#10):** AGENTS: D209 logged, D172 "refined by D209", D169's note, the C9, C11, C13 open points, the tracker row.

### For later plans

- **Plan 9 (C15):** a test run never overlaps `vite.build` (the CLI orders them, QC).
- **Plan 10 (Super-Plan R10):** C7 also depends on C4 (`RepoFiles`, `copyFixture`) and C6 (`loadTestRunners` in its CLI test); the super-plan's C7, C10 and C11 texts still mention `runTests`, D3 and module mapping (`01-super-plan.md` l.65, l.269, l.275) (R2#9).

### Skips

- Behaviour on Vitest 3.2.7 and 4.1.12 (the plan's cited versions; R2 checked 3.2.4 and 4.1.0): covered by the fixture matrix at implementation.
- R2#8, R2#12–R2#15: dissolved by QA (fresh listing instances).
- R1 2.2: the smaller additions need no decision.
