# C1 amendment A4 review (`chunk-01-monorepo-foundation.md`, the representation redesign)

> Plan 3 of the redesign's application (08b §12, 08e E7). Two zero-context reviewers, read-only: **fidelity and fit** (C1 vs 08b, D160–D199, C2, the C1 checklist in `02b-c2-b9-review.md`) and a **dogfood walkthrough** (would every planned package of this repo pass `tsshowcase check` under C1 §8?). Verdicts: A4 faithful in substance, nothing fails outright; several conventions unstated, four statements inaccurate, one pre-existing C1 bug (type tests not found). Every item discussed with the user one by one (2026-10-20). Status: **all outcomes applied** to C1 (and C2 amendment B10, D200); **plan 3 approved by the user (2026-10-20)**.

## Questions (decided by the user)

- **Q1 (option b, the user's choice):** entry files list the public API **by name** (`export { X, type Y } from './area/X.js'`), no `export *`, enforced by an ESLint `no-restricted-syntax` rule: "it makes it very explicit what is ultimately being exported, in a single place where it can easily be seen and reviewed". Claude had recommended (a), keeping internals in files the entries don't re-export. A `_` prefix never hides an export.
- **Q2 (option a):** root documents (AGENTS.md, CLAUDE.md, REQUIREMENT.md, CONTRIBUTE.md, …) stay home docs and `check` validates their links; should the verbatim REQUIREMENT.md ever trip it, a narrow `rules` override or an `ignore` of that file. Rejected: ignoring the agent files now (links to them would break); a CLI home-docs filter (new scope).
- **Q3 (option a):** this repo's root config sets `strict: ['check', 'warnings']`, so warnings fail the dogfood check locally and in CI alike (C9 writes the config).
- **Q4 (option a):** the `debug` Vitest project is opt-in (`TSS_VITEST_DEBUG=1`, set by the debug launch configs and a `debug` script outside `test:*`), so tsshowcase's own listing and runs see only `unit:*`; the user will copy this into other repos. Deferred: runner-specific options such as Vitest project selection (option b).
- **Q5 (recommended option, the user's reasoning):** this repo uses `.tsshowcase.jsonc` files only and no `directories` entries: "it doesn't require extra configuration and is therefore a good showcase of how to (simply) use the library". The user prefers the extensionless `.tsshowcase` and wants **both** it and `.tsshowcase.ts` documented in C13 as pitfalls, with the configuration each needs.

## Decision from a tangent

- **D200 (the user: "absolutely needs to be (a) - we care about what is exported, not how it is exported"):** a symbol's kind follows what the exported value is, never how it is declared: framework kinds first, constructible → `class`, callable → `function` (incl. `const f = () => …`, `const f = memoize(g)`), any other value → `constant`, no value → `type`. Belongs in the contract (C2 amendment B10, §4 `SymbolKind`, §5.4), binding for every module; C8 implements detection. Rejected: by syntax (`const f = curry(g)` would escape tests), by keyword. Makes Fix 8a unnecessary (dropped).

## Fixes (applied to C1)

- **Fix 1:** a `.tsshowcase.jsonc` creates a menu level only when it is `true`/`{}` or sets a node key; only `collections`/`kinds`/`exempt` creates no node; in `src/` it configures the root; a configured node's page shows the directory README.
- **Fix 2:** a JSON file is exporting only when its default export is re-exported from an entry; imported-only JSON and JSON `exports` subpaths (08a U9) are not; sidecars never.
- **Fix 3:** `scope: ['src']` keeps package-root files out of scope; package READMEs and licences are home docs either way.
- **Fix 4:** a README in every directory that directly contains exporting files (nested ones need their own; none without exporting files); same-base and case traps.
- **Fix 5:** entry files only re-export by name; nothing exporting directly in `src/` (a lone file would claim `src/index.test.ts`); every `exports` subpath has a `source` condition; `src/bin.ts` exports nothing.
- **Fix 6:** a collection's resources are named after its key; members' own matches count.
- **Fix 7 (pre-existing bug):** `typecheck.include` adds `*.test-d.{ts,tsx}`; build excludes list every test form, showcases and `src/compat/`; acceptance criterion 4 extended.
- **Fix 8:** (a) dropped (D200 covers it); (b) public API from dependencies gets a local declaration, never a bare re-export.
- **Fix 9:** trigger rows for `kinds`/`exempt` entries (a stated reason), showcasing components, module `static kinds`; moves update both directories' READMEs; new packages' own config; the entry's named re-export line.
- **Fix 10:** A3's stale note marked superseded; the A4 summary complete; the §4 type-test note without a decision reference; file names in their export's case; D162 linked instead of restated.
- **Fix 11:** `-lint` noted in the layout (C3, D88); additional Vitest projects per package (`unit:<pkg>:<part>`).

## Recorded for later plans (nothing changed there)

- **C8 (plan 11):** implement D200; ambient declarations (`declare module 'virtual:…'` in a `.d.ts`, e.g. `-base-vite`'s `src/client.d.ts`): symbols or not; a package with two tsconfigs; re-exports across workspace packages are external; C1 relies on 08a U9.
- **C8/C10:** hooks without a demo of their own (collection with their component, or a documented `exempt`); the `shell` opt-in format in `ShowcasingComponent.meta`.
- **C15 (plan 9):** runtime tests importing virtual modules need a mocking convention (`vi.mock`) or a reasoned `exempt`; the extra `unit:tsshowcase-base-vite:runtime` project.
- **C4, C5, C6 (plans 5–7):** stale "exported unit", "own directory", "per-unit barrels" wording (also on 02b's checklists).
- **C9:** this repo's root config (`scope: ['src']`, `strict: ['check', 'warnings']`).
- **C13:** the `.tsshowcase` and `.tsshowcase.ts` forms with their pitfalls and configuration; named re-exports; `_` doesn't hide exports.
- **Deferred:** runner-specific options (Vitest project selection).
