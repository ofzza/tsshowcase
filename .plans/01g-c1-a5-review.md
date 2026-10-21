# C1 amendment A5 review (`chunk-01-monorepo-foundation.md`, the TS template as the source)

> Raised by the user after plan 5 (2026-10-21), done before plan 6. Edits **Chunk C01** and **AGENTS** (D203, D52's status). Status: **draft applied to C1 (2026-10-21), redrafted after TP1; TP1–TP2 settled; review skipped by the user; **A5 approved by the user (2026-10-21)****.

Sources: the template at `~/projects/ofzza/.templates/ts` (last commit touching `ts/` when A5 was drafted: `e1756a1`); the research notes `01f-c1-a5-template-notes.md` (a subagent's comparison with C1, corrected after reading the `project` skill).

## Choices (discussed one by one)

- **TP1 (the user's direction, replacing the first draft):** the first draft made C1 start from a full copy through the `project` skill, with a consolidated deviations list, a copy/adapt/drop table and eight adopt-or-keep choices (debug variable, test globs, ESLint setup, tsconfig details, …). The user: "for now I just care that we are aware and don't forget about the template as the source"; no `project` skill; copy by hand what suits; what needs merging and consolidating is done interactively with the user; after the import, log what was imported directly, what was merged or changed, what was left out, and which template commit was used, so template changes can be checked and considered later. Applied: C1's baseline names the template as the source, the import step and the import log; the first draft's §0 and its adoptions were reverted (C1 is otherwise as approved); the adopt-or-keep items stay in `01f` as the import's starting checklist.
- **TP2 (option b, the user's choice):** the import log is its own file, **`TEMPLATE.md`** at the root, human-readable: which template repository and directory, which commit (and date), and what was imported and how, per file; linked from AGENTS "Project" so agents find it; a C1 trigger row covers template changes. Rejected: (a) a section in `AGENTS.md`; (c) `CONTRIBUTE.md`.

## Review

Skipped by the user's decision (A5 is small, with next to no impact on the other plans). One inconsistency Claude had flagged for a reviewer was fixed directly: the baseline's "Deviations are called out explicitly" now adds that differences not yet called out (`01f`) are settled during the import and recorded in `TEMPLATE.md`.
