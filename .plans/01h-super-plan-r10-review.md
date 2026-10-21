# Super-plan R10 review (`01-super-plan.md`, the representation redesign)

Plan 10 of the redesign's application (`08b-representation-redesign.md` §12, `08e` E7), after AGENTS, C2, C1, C3, C4, C5, C6, C7 and C15. Checklist sources: 08b §12 ("Super-plan (R10)"), `02b` "Saved for the later plans", and the "for later plans" sections of `01e`, `03c`, `04b`, `05b`, `06b`, `07b`, `15d` addressed to the super-plan.

## Choices the draft made beyond the decisions (discussed one by one, 2026-10-21)

- **Choice 1 (1/6), C8 text:** the draft wrote C8 at summary level; **the user chose a stub** (goal, "rewritten in plan 11", 08b §12 and the AGENTS open points meanwhile, depends on), since plan 11 comes right after.
- **Choice 2 (2/6), edges:** C7 → C4, C6 added (`07b`); C9 → C5 kept (the docs step's API) with C5 annotated "implemented after C9" (D194). Ok (refined by Q1 below).
- **Choice 3 (3/6), duties table:** the draft split ownership with "/" in shared rows; **the user chose separate rows**, one owner per row (the `vcs.ref`, rules, `--strict`, docs step, lazy docs and shutdown rows split).
- **Choice 4 (4/6), `-base-react` row:** the base class lists `resolveRoute` among its methods (a default incl. D204's line-anchor default, layouts may override; `runTests` dropped). Ok.
- **Choice 5 (5/6), chunk-text depth:** chunk texts carry 08b §12's items plus a few scope-level open points (C9 stderr warnings, C11 ref log and restart rules, C12 Chromium-only local runs, C13 topics, C14 schema copies and skipping `-test`); the long tail stays in the AGENTS open points. Ok.
- **Choice 6 (6/6), history:** R1–R9 kept; **the user chose to annotate each superseded item** *(R10: …)* "just in case", instead of one sentence only; the P4 row notes the redesign. Applied (30 annotations, the R10 note says so).

## Agent review (2026-10-21)

Two zero-context, read-only agents: (1) the checklists and decisions (AGENTS, 08b §12, the review files' super-plan items; stale text, contradictions with D160–D210, edges and orders); (2) the fit with the approved plans C1–C7, C15 and the AGENTS open points for C8–C14 (chunk texts, duties-table owners and gaps, package rows, edges).

### Questions (decided by the user)

- **Q1 (1/2), C5 ⇄ C9 (both agents):** C9's plan uses C5's `checkDocs` API, C5's implementation uses C9's representation and `PathNodeIndex` (D194). **(a) approved:** both directions named in the edges table: C9 "C5 (planning only: the docs step's API; C5 is implemented after C9 and wires it)", C5 "(implemented after C9: the representation, `PathNodeIndex`, D194)". Rejected: inverting the edge (contradicts wave P3), leaving it.
- **Q2 (2/2), `BuildResult` writes (agent 2):** **(a) approved:** a duties row "Writing `BuildResult.files` (D64): path validation, inclusion checks, output guard, atomic swap, marker file, module files under `tss-assets/` rejected | C11", beside the new scaffold row (F4).

### Fixes (approved together by the user, applied)

- **F1** `vcs.ref` rows per C4 §9: lazy resolution, cached per scan → C4; the first VCS URLs from `checkDocs` → C5; `ref`/`fileUrlTemplate` left out of `VcsInfo` on failure → C9 (shared).
- **F2** `module/error` row: undeclared kinds, a throwing `verify`, a reserved-namespace module issue; invalid analysis keys are thrown (D170, C2 §5.4).
- **F3** runner load failures: `tests/runner-failed` for unpinned candidates and throwing `detect`; a pinned runner failing to load is a thrown `PluginError` (C6 §11, CC2); C6 text and row.
- **F4** a row for writing `ScaffoldResult` files (C2 §5.10) → C9; C9's `init` bullet "writes the returned files".
- **F5** C2 statics: `displayName`, `rules` (own namespace, D175).
- **F6** C2 test-runner bullet: no overlap is the CLI's ordering guarantee (`07b` QC).
- **F7** C5: same-doc links marked `data-anchor` (`05b` Q3).
- **F8** C10: the router's `data-anchor` behaviour.
- **F9** C11 `build`: the one-time no-VCS warning (`unlinkedVcsLinks`, D110).
- **F10** C11 `dev`: a changed set of existing packages restarts `dev`; `checkDocs` per snapshot; the runner set and run plan frozen per session.
- **F11** C12: a `-base-vite` real-browser bullet (C15's recorded C12 items); broken and unlinked link states.
- **F12** `-base-vite` row: embeds the module's prepared data, not the `Representation` itself.
- **F13** P3 rationale: edges added after the wave was planned.
- **F14** R7 "Implementation order unchanged": annotation corrected ("still true; the order is now D194's").
- **F15** R7's D130 mention cross-references D201 (`04b` D).
- **F16** R8's "void" annotation placed on the C6 amendment; R4's D100 and R2's "three rules (D86)" annotated.

### Skipped (ok)

- `packageExists()` not named in C4/C6 texts (covered in substance) (×2); two C6 rows touching `--rule` (consistent with D207); the review file (this one); `displayName` (folded into F5); stale-term grep clean.
- C9 minor omissions (`addRoles`, printing `ConfigError`/`PluginError`, node-id stability, `not-run` reasons) and C1's pre-existing omissions (drift checks, `ci:schemas`, the `debug` project): in the AGENTS open points and C1's plan.
- Other package rows, edges and the implementation order confirmed; C11's edges complete transitively (C4 via C9, C15 via C10).

## For later plans

- **C8 (plan 11):** replace the super-plan's C8 stub with the rewritten plan's goal, requirements and dependencies (a wording edit under the super-plan drift rule).
