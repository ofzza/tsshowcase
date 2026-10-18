# C4 plan review: fresh-eyes triage

A clean-context agent reviewed every plan so far (REQUIREMENT, AGENTS D1–D102, super-plan R4, C1, C2 with B1–B3, C3, the earlier triages) with a focus on the C4 draft (round 3) and the changes made while planning it (D103–D108, the proposed B4/R5/AGENTS edits). Verdict: the draft covers every C4 requirement and open point, and the C6 seam is clean. Two findings are serious (M1: the `ignore` precedence rule disables nested `.gitignore` files below a config `!`; M2: fixtures whose content is gitignored can't be committed), and the B4/R5/AGENTS edit lists miss several stale passages. 22 findings (M1–M22). Status: **group A applied** to the C4 plan (round 4); group B resolved (M1, M2, M19–M22).

## A. Applied directly

| # | Finding | Fix (in the C4 plan) |
| - | ------- | -------------------- |
| M3 | B4 misses stale C2 text: the §5.3 comment block and `list(dir)`'s required argument; §5.4 "every non-ignored directory"; §5.5 "every non-ignored directory" and "the repo root is verified too" (false when `scope` doesn't cover it); the §4 `fileUrlTemplate` example with `{webUrl}` | All added to B4. Template placeholders are `{branch}` and `{path}` only; `webUrl` and the path prefix are baked in; a user-supplied `vcs.fileUrlTemplate` is used verbatim (it must contain any prefix itself) |
| M4 | R5/AGENTS lists miss C1 (§2 tables, §6 changesets, §5 "C4 decides where the helper lives"), the C9 open point and C1 "Recorded" (`exclude` → `ignore`), D101's status, AGENTS §1 wording, D41's `-test` exemption | A **C1 amendment A3** added to the proposed edits; the AGENTS list extended |
| M5 | Dropping out-of-scope units "before the D70 validation" hides a module returning a nonexistent directory | Order: unit directory exists → drop out-of-scope or ignored units (info log each) → the remaining D70 checks |
| M6 | Dropped units leave holes: the module's `state`, out-of-scope `analyze` issues, links to dropped/ignored targets, out-of-scope groups, tests of dropped units | Recorded for C5, C7/C10, C9 (the filtered analysis every later call receives; out-of-scope issues shown at package/repo level), C11 |
| M7 | Missing URL forms: `git+https://`, `git+ssh://`, `git://`, `ssh://` with a port | Added to §9 and the tests |
| M8 | Inherited `GIT_DIR`/`GIT_WORK_TREE`/`GIT_INDEX_FILE` (pre-commit hooks) and `safe.directory` errors | `GIT_*` variables stripped; any git failure = no git info, reason at debug level; only queries that don't read the index |
| M9 | `pnpm-workspace.yaml` without `packages` (pnpm 9/10 settings/catalog files); invalid root manifest in a single layout undefined | No `packages` key → fall through to `package.json#workspaces`; any unparseable root manifest is fatal |
| M10 | Symlinks: `list()` vs eager `directories` mismatch; "lexical only" vs C2's "resolving outside the root"; special files | `DirectoryEntry.symlink: boolean`; symlinked directories are listed (eager and `list`) but never walked; C2's boundary reworded as lexical, symlinks followed (B4); FIFOs, sockets and broken links skipped |
| M11 | Dev invalidation: `.gitignore`/`ignore`/`scope`/marker changes change whole subtrees; parent `.gitignore` files sit outside the watched tree; module memoisation escapes `AsyncLocalStorage` | Recorded for C11 (verdict changes → full rescan; watch boundary `.gitignore` files; config change → restart) and C8/C13 (module caches defeat read tracking) |
| M12 | `--scope`/`--ignore` vs config arrays: replace or append; relative to cwd or root | Recorded for C6, recommendation: `--ignore` appends, `--scope` replaces, both relative to the repo root |
| M13 | An explicit `outDir` equal to or containing the root or a package silently empties the scan | `scanRepo` throws (C4 §10); C6 validates early |
| M14 | "Patterns that match only files" is undecidable for globs; picomatch `dot`; `isInScope` really means "is scanned" | Rule dropped (the unmatched warning covers it); `dot: true`; renamed `isScanned` |
| M15 | Licence tiebreak `LICENSE` vs `LICENCE`; `reason` priority; known rule ids | Tiebreaks specified; `scan/*` and `module/error` recorded as CLI-known rule ids for C6/C9 |
| M16 | `shallow` clones bring their own `origin` and `origin/HEAD` | Removed after cloning, then `remotes`/`originHead` applied |
| M17 | `-test` as a devDependency would get a build project reference | Project references cover runtime dependencies only |
| M18 | `.gitignore` files inside a config-re-included directory | Covered by the M1 decision |

## B. Needs the user's decision

| # | Finding | Recommendation |
| - | ------- | -------------- |
| M1 | "Deepest level with a config match decides" lets `!dist/vendor/` switch off every nested `.gitignore` below it (`*.map` inside `dist/vendor/` stops applying); "level" undefined; a `.gitignore` `!` under a config-ignored directory unaddressed | ✅ **Applied** (the user agreed): direct matches decide, otherwise inherit from the parent; config beats `.gitignore` on the same path; a config `!` may revive a path under an ignored parent, a `.gitignore` `!` may not (git's rule) |
| M2 | A fixture's `.gitignore` also applies to this repo's git, so gitignored fixture content is never committed (CI lacks it); this repo's root `.gitignore` can hit fixture `dist/`/`showcase/`; `copyFixture` copies whatever is on the local disk | ✅ **Resolved by the user's proposal (D109):** a `.tsshowcaseignore` alias (lines appended to the same directory's `.gitignore`, git's rules over the combined list); fixtures use it; `.gitignore` tests are crafted so no needed file is gitignored (`copyFixture` `files` option); this repo's `.gitignore` catching fixture content is fixed with `!` rules (C1 amendment A3). Copying the local disk is fine, like any other test (the user's point) |
| M19 | Parent `.gitignore` files up to the nearest `.git` can ignore the whole root (a dotfiles repo in `$HOME` with `*`, a parent's ignored `tmp/`) | ✅ **Applied** (the user's call, going further than the recommendation): only `.gitignore` files at or below the root apply; no upward search for them. A subdirectory or single-package run stands on its own |
| M20 | One unreadable directory, or a file vanishing mid-walk in `dev`, is fatal | ✅ **Applied:** `scan/unreadable` warning and skip; `ENOENT` mid-walk skipped silently |
| M21 | `{ ignored: true }` reaches `.git` (`.git/config` may hold credentials); `node_modules` reads become watch targets | ✅ **Applied** (the user's call): `.git` and `node_modules` are unreachable through `RepoFiles`/`DirectoryInput` in every mode |
| M22 | Windows, case-insensitive and NFD filesystems not addressed | ✅ **Applied:** POSIX tested, Windows normalised best effort, case-sensitive matching, code-unit sort, no normalisation; in the CLI README |

## Round 2: second fresh-eyes review

A second clean-context agent reviewed the same material after round 1's resolutions (C4 round 5, D103–D109). Verdict: the M1–M22 resolutions are in place with three stale spots; the new semantics have one serious hole (N1: a config `!` can make a path exist under a directory that doesn't); the VCS prefix can mix sources (N9); the edit lists miss a few stale passages. 17 findings (N1–N17). Status: **resolved** (C4 round 8): group A applied; N1, N9 (D110, with optional `vcs.host`), N17 decided by the user.

### A. Applied directly

| # | Finding | Fix (in the C4 plan) |
| - | ------- | -------------------- |
| N2 | "Ignore files inside an ignored directory are never read" contradicted table row 2 | An ignore file is read if and only if its own directory is not ignored |
| N3 | "Matches directly" had no implementation primitive (the `ignore` package folds parent checks in); directory-only patterns need the path's type | Defined explicitly; a thin matcher; `lstat` for type; `dist/**` documented; a differential test against `git check-ignore` |
| N4, N5 | Q4's closing line and the C8 "Recorded" entry still had pre-M5 / pre-M2 wording | Reworded |
| N6 | B4 dropped `symlink` from `DirectoryEntry`, said `ignored: true` "sees everything", and didn't carry the lexical boundary into §5.3 | B4 corrected |
| N7 | Following symlinks was justified with `node_modules`, which is unreachable | Justification replaced |
| N8 | The eager listing wasn't tracked; `invalidate` didn't drop parent listings | Both added, with tests |
| N10 | `copyFixture` `files` couldn't place a `.gitignore` above the scan root | Keys relative to the git repo root when `git.root` is set |
| N11 | `copyFixture` didn't strip inherited `GIT_*` variables | Stripped |
| N12 | The `scope-unmatched` hint suggested `!` for hard ignores; `reason: 'gitignore'` also covered `.tsshowcaseignore` | Hint per reason; reason renamed `'ignore-file'` |
| N13 | This repo's `ignore: ['fixtures/']` is unanchored | `/fixtures/` |
| N14 | `list` of an ignored directory without the option was unspecified | Rejects as missing |
| N15 | Edit lists missed super-plan C9's "every non-ignored directory", three C1 §5 passages, AGENTS §1's second mention | Added to R5, A3, the AGENTS list |
| N16 | `scope` (root-anchored globs) and `ignore` (gitignore syntax) read paths differently | Documented; the unmatched hint mentions `**/src` |

### B. Needs the user's decision

| # | Finding | Recommendation |
| - | ------- | -------------- |
| N1 | A config `!` revives a path while its ancestors stay ignored (`dist/vendor` scanned while `dist` doesn't exist); `!dist/vendor/a.js` gives a file in a nonexistent directory; a slash-less `!*.generated.ts` revives files inside every gitignored tree and forces the walk into all of them | ✅ **Applied** (the user agreed): M1's extension dropped. One rule for all three sources: no `!` revives a path under an ignored parent; config only adds priority; re-including inside an ignored directory uses git's idiom in `ignore`. The walk never enters ignored directories |
| N9 | The path prefix comes from git while `webUrl` may come from `package.json#repository`; a root inside an unrelated repo, or a fork, gets broken links | ✅ **Approved by the user (D110), after discussion:** no git at all. `webUrl` and the prefix come from the root `package.json#repository` (parsed with `hosted-git-info`; prefix = `repository.directory`), overridden by config `vcs.webUrl` / `vcs.directory` / `vcs.fileUrlTemplate` and matching flags; links always use `HEAD`; nothing known → no host links, one `build` warning. Revises D24, D105. Optional `vcs.host` kept (the user's choice) for self-hosted instances. (Claude first proposed a conditional git prefix; the user questioned it, then considered mandatory config; option (a) was approved) |
| N17 | `analyze` issues whose `dir` is out of scope are still shown (M6), against D101's "without an issue" | ✅ **Applied** (the user agreed): dropped with an info log line, like units; issues without a `dir` kept. Supersedes M6's "shown at package or repo level" |
