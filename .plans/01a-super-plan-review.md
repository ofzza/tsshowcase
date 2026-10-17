# Super-plan review: fresh-eyes triage

An agent with no prior context reviewed REQUIREMENT.md, AGENTS.md and the draft super-plan.
Its verdict: the docs are largely self-explanatory, with the main weak spot being who owns what across the module, the CLI and the runner.

## Applied directly to the super-plan (no decision needed)
- The C4 goal now says it covers only the repo/package/directory layer; units come from `analyze`.
- C8 now also ships the loadable `-module-react-default` class with its Node-side methods. Without it, the `check`/`init` slice (D29) had nothing to load.
- C8 maps `exports`/dist back to source, and uses the repo's own TypeScript.
- Scaffold templates keep showcase files out of the user's build.
- Added edges and fixed waves:
  - C8 → C4 (fixtures);
  - C10 now depends on C2, not C7;
  - P4 merged into P3, with C8 moved to its own wave;
  - the ASCII graph is replaced by an edge table.
- The C2 plan stays open until C3 is approved (catalogue circularity), and C3's implementation extends `-api`.
- C2 additions: a repo-level verification slot, and tolerance of unknown fields in module outputs.
- C3 additions: theme persistence with no flash; theme scoping and extension; CSS isolation between the shell and user showcases; an ESLint ban on inline styles.
- C4 additions: `.git/info/exclude`; always ignore the output dir; non-git behaviour; a fallback for default-branch detection.
- C5: a heading-anchor scheme compatible with hash routing.
- C10: React dedupe also covers module runtime files and peer auto-install; `server.fs.allow`.
- C11: guard against writing into a non-empty output dir (stdblox has `showcase/`); `dev` re-analyses when files are added or removed.
- AGENTS.md: D31 now clarifies D13 (the `-api` version = the interface version); the D3 inversion of "tests come from the CLI" is noted in the assignment summary; the stage status mentions D26–D33.

## Needs user decisions
1. ✅ D34. Test ownership: does the module pick and run the runner (D3), or is the runner a user-selected plugin that the CLI calls?
2. ✅ D35. Route ownership: who maps unit → URL for link rewriting at CLI time?
3. ✅ D36 (nesting allowed) and D38 (type-only units need a README only). Unit definition: is a unit one directory or one symbol? Are type-only exports units? Can units be nested?
4. ✅ D37 (showcases required per kind, so pure TS needs no React). Pure-TS libraries: where does React come from when the repo has none? What does a showcase for a non-React unit look like?
5. ✅ D39. Syntax highlighting (C5): keep it (class-based) or drop it? It was not in the requirement.
6. ✅ D41. Dogfooding: do this repo's own packages follow the unit convention from C1 onward?
7. ✅ D40. Lazy file accessors vs "the list of files and their contents" (the requirement wording).

## Deferred to chunk plans (recorded there as requirements or questions)
- C2: schema technology; whether the JSON report shape is contract or CLI detail; per-package config and exclusion in workspaces.
- C3: how the default is picked when there are more than two themes; whether user showcase components are bound by the theme rules.
- C4: whether workspace packages are scanned from the root or per package.
- C7: supported Vitest range; repos with no Vitest or with Jest; Vitest projects config; tests outside the unit dir; two Vite versions in one process.
- C8: `init` semantics (new unit vs filling missing stubs; adding the export to the entry).
- C10: supported React range; workspace cross-imports resolving to a stale `dist` (alias to source?).
- C13: document that `check` executes config and custom modules (untrusted PRs in CI).

## Final consistency review (after D48)

A full pass over REQUIREMENT.md, AGENTS.md and the super-plan. Fixed:
- **AGENTS.md formatting:**
  - blank lines inside the decisions table broke it into several tables;
  - the assignment-summary "changed by" notes were mis-indented and incomplete (they now cover D37/D38 for `check`, D40 for `verify`, D16 for `dev`/`init`).
- **Stale notes in AGENTS.md:**
  - D14's package list;
  - D16/D18 still referred to Vite (now module-owned, D43);
  - D20's "probably a package or a base class" (resolved by D26/D48);
  - D21's rules (refined by D37/D38);
  - D26's "Vite contribution" (now the React wrapper over `-base-vite`, D48);
  - D30's "release scope still being discussed" (settled by D33);
  - stage-status text.
- **Naming conflicts:**
  - D34 `runner` vs D42 `moduleOptions.testRunner` → unified on `testRunner`;
  - `ctx.onUpdate` (D42) vs `onDataUpdate` / `onTestUpdate` (D43, C15) → unified on `onDataUpdate` + `onTestUpdate`.
- **C2:** output tolerance still listed "Vite config", which is no longer in the contract (D43). Added: source-agnostic package info (requirement: "later other repo types"); root vs package README/licence and the single-package collapse; units that are both page and parent (D36).
- **C5:** "anchors left untouched" contradicted the hash-routing anchor scheme. Heading anchors now go through `resolveRoute(target, heading)`.
- **C7:** "maps results per unit" contradicted D42 (the module maps). The runner now returns the raw hierarchy.
- **C8:** kind classification lacked `constant`, which D37 needs.
- **Gap: no chunk implemented the base class `runTests`.** Added it to C10, plus the C10 → C7 edge.
- **C10:** nested-unit navigation (D36); showcases of `-base-react`'s own components (D41). The switcher UI stays in C10, while persistence and no-flash init move to C15.
- **Gap: fixture isolation from the workspace** (raised in the first review but never assigned) → added to C1.
- **C11:** open point on strict mode vs "tests unavailable" and on test-result caching, recorded for the chunk plan.
- **Pitfalls (AGENTS §6):** each one is annotated with the decision or chunk that resolves it.

No remaining conflicts between decisions. Open points are all chunk-level and listed under "Deferred to chunk plans" above, plus the C11 note.
