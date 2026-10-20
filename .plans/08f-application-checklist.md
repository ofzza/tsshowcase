# Draft application checklist for the representation redesign

> Status: **DRAFT, not approved** (2026-10-20). Produced by a clean-context "cold-start" reviewer as a dress rehearsal of applying `08b-representation-redesign.md` (revision 3) to the plans (see `08e-redesign-review.md`). It is a starting point for the session that applies 08b, to be corrected against 08b's final revision; 08b wins wherever they differ. Proposed decision ids and texts are suggestions only.

## Order and ids

- Order: AGENTS (decisions, open points) → C2 → C1, C3, C4, C5, C6, C7, C15 → super-plan R10 → the C8 rewrite; each re-approved per the process (scope changes), committed after approval.
- Amendment ids (proposed): C2 **B9** (replacing the void B9 of the C8 round-2 draft), C1 **A4**, C3 **C3-A3**, a "redesign amendment" in C4/C5/C6/C7/C15, super-plan **R10** (replacing the draft's void R10).

## Proposed decisions D160–D195

| ID | Decision | Revises / replaces |
| -- | -------- | ------------------ |
| D160 | Principles: stage interfaces first; the user's own, mainly new repos; presentation belongs to the module | – |
| D161 | RQ1: every public symbol needs representation (its file's node or a configured node) | D36 |
| D162 | RQ2: kinds named explicitly; CLI requirements table; module-declared kinds via static `kinds` | D37, D38 |
| D163 | RQ3: docs are markdown associated with the symbol; a directory README documents every exporting file in it | D21 (README part) |
| D164 | RQ4: `--strict` parts; `build` always checks; removed `--no-check`, `check`, `--fail-on-test-failure`, `failOnTestFailure`; `check --strict=warnings` | D8, D76 |
| D165 | RQ5: in-scope only; package-relative `scope` | D101, D107, D112 |
| D166 | RQ6: CLI check and test runners; module enriches; per-file `verify` | D2, D21, D40 |
| D167 | RQ7: no optional positionals; `build --out` | D93 |
| D168 | The pipeline (08b §3) | – |
| D169 | Scan: typed in-scope tree per package; stops at nested packages; files before subdirectories; directory configs in scope only | 08a U16 |
| D170 | `AnalysisResult` shape with `meta` passthrough, `issues`, `inputs`, `state` | D5, D68 |
| D171 | Symbol attachment and TypeScript rules (08b §4.1) | – |
| D172 | CLI-owned test runners (candidates, pinning, all detecting, listing intersected with scope, failures, short-circuit) | D3, D34, D42, D74, D75, D135, D136; D9/D114 |
| D173 | Association levels and rules (08b §4.3) | – |
| D174 | The check rule; one issue per symbol and type; listing vouches | – |
| D175 | Per-file `verify` and dev invalidation | D2 |
| D176 | `RepresentableNode`; opaque ids; final order | D78 (groups) |
| D177 | Tree building; stable root id with merged configuration | – |
| D178 | Issue attachment; `Representation.issues` | D70 |
| D179 | Weak associations are presentation only | – |
| D180 | Coverage and per-node test summaries are module concerns | – |
| D181 | Link resolution (08b §7.7) | D35, D67, D73, D78, D121 (D124 kept) |
| D182 | Doc-id-prefixed element ids | D127, D128, H36 |
| D183 | Homes with roles; module selects | D19 |
| D184 | `Representation.tests`; shared revision; merge rules | D68, D141 |
| D185 | `Representation` replaces `ShowcaseData`; lazy docs; assets only for rendered docs | D23, D43, D120, D125 |
| D186 | Directory configs and `directories` | D154, D155, D156 |
| D187 | Kind overrides | D158 |
| D188 | Collections | – |
| D189 | `exempt` | – |
| D190 | Config keys: `testRunner`/`testRunners` CLI keys, rule-id patterns, `--rule`, removed keys, package-config keys | D59 |
| D191 | Packages always present | D72 |
| D192 | Implementation order (C7 before C9) and edges | D29 |
| D193 | Showcases per (file, export); `shell`/title via `meta`; loaders | D61, D96, D100, D140, D149 |
| D194 | Vitest 3/4 inline roots documented only; single-file rule kept | – |
| D195 | `init` re-expressed | D150 |

Then: statuses of existing decisions per 08b §11 (each "replaced / changed by D1xx"); AGENTS §1 (assignment summary annotations, "Architectural shifts"), §2 (tracker, "Current focus", open points per 08b §11.4 plus new items, a "Deferred" row from 08b §13), §4 (L3 resolved, L2 → C11), §6 (per-directory pitfalls superseded).

## Per plan

- **C2 (B9):** design principles (2: `Representation`; 4 removed; 6: CLI owns check, associations, runners); §1 dogfood layout under the new rules; §3 `SymbolKind`; §4 data model replaced (`Representation`, `RepresentableNode`, `RepresentedItem`, `Association`, `ShowcaseAssociation`, `DocRef`, `Tests`; keep `PackageInfo`, `VcsInfo`; render-result type; tree building; tests; homes); §5.1 statics `testRunners`, `kinds`; §5.2 methods (`runTests` removed); §5.3 `RepoFiles` stays, `DirectoryInput` removed, scanned tree and `FileInput`, `AnalyzeContext`; §5.4 `AnalysisResult` and attachment rules, new D70; §5.5 per-file `verify`; §5.6 `resolveRoute` target object, links subsection; §5.7 replaced by a tests note and `TestsUpdate`; §5.8 `BuildContext.data`, `docs.render/read`, `onDataUpdate`, `onTestUpdate`, `onDocsUpdate`, assets rule; §5.9 call order from the pipeline; §5.10 `ScaffoldContext`; §5.11 `testRunner` paragraph removed; §6 `listTestFiles`, process restore around listing; §8 rules 6 (prefixed ids), 7 (`shell` via `meta`), embedding of `Representation`; §9 issue model (rule namespaces, patterns, `--strict`); §10 config sketch; §12 guards (`AssertJsonSafe<Representation>`, `<TestsUpdate>`, compat guard replaced); §13 glossary; acceptance criteria; recorded points; open questions annotated.
- **C1 (A4):** §8 new dogfood layout (one exporting file per concept, README per directory, tests/showcases named after files; barrels and `_` prefixes unnecessary; build-tsconfig exclusions stay); §10 trigger rows; §5 and recorded C9: `ignore: ['/fixtures/']` dropped (fixtures are outside every package); glossary seed.
- **C3 (C3-A3):** `tssData.kind` = symbol kinds; broken-link state attribute; association/node-issue attributes now or with C10; "Unit info" renamed.
- **C4:** seam (`ScanOptions.scope` per package, typed trees, collected directory-config files, `toRepoInput`); §5 package-relative scope, per-entry unmatched warning, no package drop, D106 paragraph removed; §6 per-package trees, nested packages, files before subdirectories; §7 `DirectoryInput` removed, tracking kept; §8 homes with roles, README forms; a new section on directory configs; recorded points.
- **C5:** §3 doc-id prefixing; §4.1 replaced by 08b §7.7; §4.2 fragments; §4.3 permalink owner; §4.5 `<a>` without `href`; §5 callback `(target) => string`; §8 API (`LinkTargets` → representation, lazy per-doc render, `checkDocs` over every `DocRef`, assets per render); §8.2 hierarchy helper deleted; recorded points.
- **C6:** `ResolvedConfig` (`strict`, `rules` patterns, `--rule`, `--out`, `loadTestRunners`, normalised directory configs); §3 argv normaliser and `--rule` parsing; §5 schemas (root, package, `directory-config.schema.json`, `defineDirectoryConfig`); §6 precedence table; §7 per-package merging; §8.1 module `testRunners` resolution; §11 runner loading rewritten; §12 rule ids; §13 "did you mean" for removed keys and the old positional.
- **C7:** detection unchanged, several detected no longer an error; `listTestFiles` via `globTestSpecifications()`, process guard around listing, Vitest 3/4 inline-root caveat; §10 cwd test scoped; README: `testRunner` is a CLI key; recorded points move to C9/C11.
- **C15:** `prepareData(Representation)`; showcases module keyed by file, `loadShowcase(file, exportName)`; `applyTestUpdate` over `Representation.tests`; one revision; docs delivery (see 08e E2).
- **Super-plan (R10):** revision note; package table rows; chunk texts (C1, C2, C4–C11, C12–C15); CLI-duties table rewritten; edges (C9 → C7 added, C10 → C7 dropped); implementation order.
- **C8 (rewrite):** `ReactModuleBase.analyze` → `AnalysisResult` (TS selection D152; 08a U9–U13; attachment and kind rules incl. the syntactic fallback; `meta`); showcasing components with `meta`; per-file `verify` (showcase format, `headless/*`); `react/showcase-in-build` as an analysis issue; `scaffold` per 08b §10 (+ U7, U8/U18); options (`homeDocs`, markers; no `testRunner`); fixtures re-derived; claims, units, groups, `unitRoot`, `showcase: false` removed.
