# C5 plan review (fresh eyes, round 1)

> A clean-context agent reviewed the C5 draft (round 2) against every approved plan, the decisions log and the amendments C5 proposes (C2 B6, C3-A1, super-plan R7). Read-only; the footnote claims were checked against installed `mdast-util-to-hast` 13.2.1. Status: **resolved**; applied in the C5 plan, round 3, approved 2026-10-19. Decisions: R10 → D124 (a non-issue: brokenness never depends on routes; replaces options a–c), R25 → D125 (option b), R20 → D126, R1/R2 → D127, R6 → D128 (author anchors passed as their real element id), R9 → D129 (copying kept after the user weighed absolute VCS URLs; real path inside the root and not ignored), R8 → D130, R7 → D131.

**Verdict:** C5 matches the approved plans in most places. The problems are at the edges: footnote and id handling in remark-rehype and the sanitiser; `checkDocs` not seeing routes; B6 not stating module duties (`dev` assets, URLs relative to the site root, rule 2 and README images); knock-on edits that R7 doesn't list.

## A. Needs a decision (recommendation given)

| # | Finding | Recommendation |
| - | ------- | -------------- |
| R10 | `checkDocs` has no `resolveRoute`, so group `routed` and the docs' own routes are unknown in `check`; anchors into or within group READMEs are classified differently in `check` and in `build`. A check-only module may lack `resolveRoute` (D63). | **(a)** `check` calls `resolveRoute` when the module has it, otherwise treats groups as route-less; "one classifier" holds given the same `LinkTargets`. (Alternatives: (b) never validate anchors into or within group READMEs; (c) `check` requires `resolveRoute`.) |
| R25 | **Blocker for B6:** the `dev` duty to serve doc assets is not in the contract; third-party modules would show broken images. | **(b)** the CLI mirrors `build`: in `dev` it copies the current assets into a CLI cache directory (`tss-assets/` layout, rebuilt on data updates so old hashes disappear) and passes `DevContext.docAssetsDir`; the module serves that directory statically at `tss-assets/`. Path safety stays in the CLI. (Alternative (a): every module serves `Doc.assets` itself, the duty written into B6.) |
| R20, R36 | D119 strips author `class`, so C3 §2.1 rule 3, C2 §7 and D86 ("own classes are useful … in raw HTML in READMEs") become false. | **Keep stripping** (portable READMEs, GitHub parity) and fix the stale text: own classes are useful in shell-mode showcases only. |
| R1, R2 | Footnote ids come from the **label** (`fn-<label>`, `fnref-<label>[-n]`), and remark-rehype plus the sanitiser would prefix them (`user-content-`, possibly twice). | **Bare ids:** `remarkRehype({ clobberPrefix: '' })`, plus a post-sanitise pass that strips the prefix from footnote ids and their `aria-describedby` only. Tested, including labels like `[^Note 1]`. |
| R6 | Author anchors (`<a name="x">`, `<div id="x">`) are valid `#x` targets on GitHub, but C5 would report them as broken. | **Valid targets.** Author ids keep their `user-content-` prefix in the HTML; a `#x` link that matches one passes the real DOM id (`user-content-x`) as `heading`, so C10 just looks the id up. B6 wording: `heading` is any element id in the doc's HTML that C5 produced or kept. |
| R9 | Assets follow symlinks (C4 §7 is a lexical boundary), so a committed `docs/x.png -> ~/.ssh/id_rsa` would be published, which is dangerous in CI on untrusted PRs. | Assets only: the **realpath must stay inside the repo root**, otherwise `docs/broken-link`. Tested. |
| R8 | "Case-sensitive" matching vs `lstat` on macOS/Windows: links, assets and issues differ per machine, which breaks determinism. | **Enforce exact case** with a (tracked) parent `list`. Tested with a fake file layer. |
| R7 | starry-night's `pl-*` classes are coarse: `number`, `type`, `operator`, `punctuation` and `property` would rarely or never be emitted distinctly. | **Document** the mapping table in the plan and the catalogue docs, including the tokens starry-night never produces (kept for other highlighters); derive the emitted-class list from starry-night's shipped CSS; ship the `common` grammar set. C3 unchanged. |

## B. Accepted fixes (wording and consistency, no decision needed)

| # | Fix |
| - | --- |
| R3 | Footnote and task-list output listed in §2.1: `sr-only` → `tss-visually-hidden`, other generated classes (`footnotes`, `data-footnote-backref`, `contains-task-list`, `task-list-item`) stripped; the footnote label `h2` gets no slug, permalink or `Doc.headings` entry. |
| R4 | The sanitiser keeps `code` `language-*`; `highlight/` reads it, then removes it. |
| R5 | Raw-HTML headings: the slug id replaces an author `id` on the heading itself; the author id moves to an inner empty `<a id>` (still a valid anchor, R6). |
| R11 | Markdown licences (`LICENSE.md`, D71) are checked and rendered like other markdown docs, without an owner (§4.3). |
| R12 | The README rows of §4.1 require the README file to exist; the directory rows still match. |
| R13 | `srcset`: only the bad candidate is dropped. The allowed media attributes are listed explicitly; `autoplay` is not allowed. |
| R14 | `data:` media URLs are removed by the sanitiser (stated, not "untouched"). |
| R15 | §2.2 reworded (the default schema already allows checkboxes); one sentence: README markup is user content, outside D46, like showcases. |
| R16 | Alerts: the marker is case-insensitive, alone on the first line, top-level blockquotes only; `role="note"`; tested. |
| R17 | `Issue.line` is best-effort for links inside raw HTML. |
| R18 | Out-of-root links in a monorepo subdirectory (`vcs.directory`) are a documented known false positive in v1. |
| R19 | Goal: group READMEs are found by C9's hierarchy helper. |
| R21 | `data-language` becomes `tssData.language` (open values) in C3-A1. |
| R22 | A **C6 amendment** approved with C5: the CLI-known rule ids gain `docs/broken-link`, `docs/broken-anchor` and the `docs/` namespace. |
| R23 | A one-line **C4 note** in R7: binary reads in `files/`, tracked as reads, covered by C4's tests. |
| R24 | B6 also updates C2 §5.6's link mapping ("media → doc assets, missing/ignored → unwrapped") and the `Doc.html` comment; D73 "refined by D120, D121", D39 "refined by D118". |
| R26 | B6 amends rule 2: "… unless the user's own showcase code or README content (external images, badges) makes them". |
| R27 | B6 rule 6: URLs in doc HTML are relative to the site root; a module inserting doc HTML into a document elsewhere sets `<base>` or rewrites them. |
| R28 | B6 rewords `BuildResult.files` and rule 1: "the site, except doc assets, which the CLI adds"; `DOC_ASSETS_DIR` in `inclusion/`. |
| R29 | R7 adds dependency edges C9 → C5 and C5 → C4 and updates the P5 wording (planning order unaffected). |
| R30 | R7: C9 reserves the docs step and its place in the report; C5's implementation wires `checkDocs` in and adds a `check` integration test on a fixture. |
| R31 | R7 lists the AGENTS open-points updates for C6, C9, C10, C11, C12, C13 and C15, and moves the super-plan C11 "Group READMEs (D78)" bullet and C2's "Recorded for later chunks" group-README line to C9. |
| R32 | R7 adds to C12: a README image in `build` and `dev`, footnote and cross-doc anchor scrolling, themed alerts. |
| R33 | C3-A1 also covers default styles for `img`/`video` (`max-width`), `kbd`, `details`/`summary`, task-list checkboxes, footnotes and `picture`, plus the alert token picks (note → accent, tip → passed, important → accent, warning, caution → failed; final picks at implementation). |
| R34 | C11 note: the "no absolute build-machine paths" output check skips doc assets (user content, binary). |
| R35 | The `dev` asset directory is rebuilt from every data update (folded into R25 (b)). |
| R37 | C10/C13 note: ids are unique per doc only; modules showing several docs on one page scope or prefix them. |
| R38 | Alerts, `video`/`audio` and near-match hints are marked as candidates for the v1 cut-line review (K21, D98). |
