# C1 amendment A5: research notes (template baseline)

> Notes for **Chunk C01 amendment A5**, to be planned after plan 5 (the user, 2026-10-21). Not a decision record: every item below is a candidate, to be settled with the user one by one. From a read-only subagent review of `~/projects/ofzza/.templates/ts` against C1 (2026-10-21), with two corrections checked afterwards.

## The idea

C1 already names the template as the baseline (D52, C1 "Baseline"). A5 would make the implementation **start from** it, so that the user's projects share one lineage: copy, adapt, apply the listed deviations, record the template commit.

- Template: `~/projects/ofzza/.templates/ts` in the git repo `~/projects/ofzza/.templates` (`https://github.com/ofzza/.templates`); last commit touching `ts/`: `fb2dfd0` (2026-10-20); `c9a0c88` (same day) made the debug project opt-in.
- Copy mechanism: the `project` skill (`.templates/.claude/skills/project/SKILL.md`, identical to `~/.claude/skills/project/SKILL.md`; arguments `new`, `ts`) copies the committed content of `ts/` into the working directory; its `git init` / "Initial commit" step runs only outside a git repo, so not here. Caveat: it would overwrite `AGENTS.md`, `CLAUDE.md`, `README.md`; those are merged by hand (the template's AGENTS sections into ours, never replacing the decision log).

## Compatibility (no blockers)

Single package → workspaces (structural); `bundler` → `nodenext` (stated deviation, C1 §3); TypeScript `^5.9.3` → 6.0.x (D50; not labelled as a deviation in C1); `engines` added by C1 although the template says not to (deliberate, C1 §11, D49); `prepublishOnly` and `PUBLISH.md` deferred to C14; Node `[22, 24]`, `@types/node ^22`, ESM, MIT/ofzza and the docs split identical.

## Drift (template vs C1), each to adopt or keep with a reason

- Vitest config `vite.config.ts` (template) vs `vitest.config.ts` (C1).
- Debug env var `VITEST_DEBUG` (template) vs `TSS_VITEST_DEBUG` (C1 A4).
- C1's npm `debug` script with an inline env var; the template deliberately has none (not portable to Windows).
- Test globs: template `.js/.jsx/.ts/.tsx` and typecheck `test-d` + `spec-d`; C1 `{ts,tsx}` and `test-d`; debug project also `*.debug.{ext}`; `.gitignore` `*.debug.test.[jt]s(x)` vs C1 `*.debug.test.ts`.
- `test:unit --passWithNoTests` in the template (and its launch configs, named with "TS: " prefixes); C1 omits it.
- ESLint: template `@eslint/compat` `includeIgnoreFile`, `globals` (browser + node), `no-undef: off`, a `files: ['**/*.{js,jsx,ts,tsx}']` entry; not in C1.
- tsconfig: template `esModuleInterop`, `rootDir`, `tsconfig.build.json` extending `tsconfig.json`, `include` with `.tsx`/`.json` (untyped `.tsx` assertions otherwise pass silently); C1 adds `verbatimModuleSyntax`, `declarationMap`, `sourceMap`, a separate `tsconfig.base.json`, unlabelled.
- The template's `start` script dropped by C1 without saying so; `.nvmrc`, `.editorconfig`, `.vscode/settings.json` added by C1, unlabelled.
- C1's "as the template" for typed linting of `packages/*/src`: the template is type-aware for `src/*.ts` only.

## Candidate copy / adapt / drop table

- **Copy:** `.prettierrc`, `.vscode/extensions.json`, `.github/dependabot.yml` (fixture entries later), `LICENSE` (root and every package).
- **Adapt:** `.gitignore`, `.prettierignore`, `.github/workflows/ci.yml` (fixtures install, lockfiles in the cache key), `.vscode/launch.json` (`unit:*`), `eslint.config.js` (type-aware `packages/*/src`, `fixtures/**`, `*.d.json.ts`, the entry-file rule), `vite.config.ts` → root Vitest config with one `unit:<pkg>` per package, `tsconfig.json`/`tsconfig.build.json` → base + root + per-package, root `package.json` (private, workspaces, scripts, TS 6.0), `src/index.ts`/`index.test.ts` → package skeletons and the CLI `bin.ts` guard, `README.md` → root stub and package README structure, `CONTRIBUTE.md` (+ glossary), `AGENTS.md` (merge).
- **Drop:** `CLAUDE.md` (ours exists), `package-lock.json` (regenerated), `PUBLISH.md` (kept as C14's source), the `hello` sample and `start` script.

## Candidate sync rules

- AGENTS records "based on `.templates/ts` @ `<sha>`".
- Trigger-table row: when the template changes, diff `<sha>..HEAD -- ts/`, apply or explicitly reject each change, bump the recorded sha.
- Improvements that originate here (`ci:readme-samples`, `ci:links`, type-test work) may be offered back to the template.
