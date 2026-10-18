# Prototype spikes before wave P3 (checkpoint review K17/K20)

Instead of recording "prototype later" gates, the user had two throwaway prototypes built by subagents during planning (2026-10-18). Code lives in Claude's scratchpad only, outside the repo, and is never merged. This file records what they proved and what the plans must change.

## Spike 1: the showcased repo's own Vitest in-process, next to the CLI's Vite 8 (K20)

**Verdict: viable.** The same runner code worked unchanged on **Vitest 3.2.7, 4.1.12 and 5.0.5**, with the fixture on Vite 6/7 (esbuild) or a deduped Vite 8, alongside the CLI's own Vite 8.3 (Rolldown) dev server in the same Node 24 process.

**What works:**

- Resolve from the repo: `createRequire(path.join(root, 'package.json')).resolve('vitest/node')`, then `import(pathToFileURL(entry).href)`; no `chdir`, env vars or pool settings. The runner package has **no `vitest` dependency** (a devDependency for types only is fine).
- Run: `createVitest('test', { root, watch, reporters: [reporter] }, {}, { stdout, stderr })` then `vitest.start(filters)`; reporter hooks `onTestRunStart`, `onTestModuleEnd` (per file, used for streaming), `onTestRunEnd(modules, unhandledErrors, reason)`; stop with `vitest.close()` (also mid-run, clean, no hanging handles).
- Mapping `TestModule` → C2's raw hierarchy: `moduleId` (relative to `root`), `project.name`, `state()`, `diagnostic().duration`, `errors()`, children (suite `state()`/`children`; test `result()`, `options.mode === 'todo'`); Vitest's `pending` = running.
- Watch: `start()` resolves after the first run; each rerun fires `onTestModuleEnd` per changed file, then `onTestRunEnd`. Latency 100–315 ms; cold run about 1 s.
- **Two Vites in one process:** no conflicts. Vitest's Vite listens on no port; the CLI's dev server, HMR websocket and watcher kept working through every rerun; start order doesn't matter; repeated runs in one process work.
- Failing tests never throw; a syntax-error file is a failed file with a file-level error and no children; a broken/missing config makes `createVitest` throw (the runner returns `status: 'error'`); `*.test-d.ts` appear as ordinary files; `projects` configs at the repo root are picked up, each module carrying its project name.

**Problems found, with verified workarounds:**

1. **`process.env` mutation:** `NODE_ENV=development` (and `MODE`, `DEV`, … on 3.2/4) leak after a run; a later `vite build` produced a dev build. → The runner snapshots and restores `process.env`; C15's build sets `mode`/`NODE_ENV=production` explicitly.
2. **`process.exitCode = 1`** when tests fail, which would break D8 (`build` must not fail on failing tests). → The runner saves and restores `process.exitCode`.
3. **No test files** throws `VITEST_FILES_NOT_FOUND`. → Map to `completed` with `files: []`.
4. Config-load errors print straight to stderr (bypassing Vitest's stderr option). → Cosmetic; optionally intercept `console.error` during `createVitest`.
5. Paths are relative to `root`. → The runner needs the repo root to produce repo-relative `file`s.
6. **The same file can appear once per project.** → Result identity is `(file, project)`.
7. **A TS syntax error anywhere makes type tests silently pass**; the real errors only appear in `unhandledErrors`. → Surface `unhandledErrors`.
8. Vitest has no public "file removed" event (only the internal watcher). → The CLI's own dev watcher is the reliable source of deletions; the runner's `removed` is best-effort.

**Plan changes (to approve):**

- **C2 §6 (contract):** `TestFileResult.project?: string`, result identity `(file, project)` (updates and `removed` keyed the same); `RawTestReport.unhandledErrors?: TestError[]`; `TestRunOptions.repoRoot` (repo-relative paths) and `files` documented as substring filters; contract requirements: runners restore `process.env` and `process.exitCode`, and "no tests found" is `completed` with no files. **C2 §5.7:** deletions come reliably from the CLI's dev watcher after re-analysis; the runner's `removed` is best-effort.
- **C7:** implement on `createVitest` + the reporter API above; `detect()` checks that `vitest/node` resolves from the repo root; never assume where `vite` comes from (Vitest 5 makes it a peer). **Supported range: `^3.2.0 || ^4.0.0 || ^5.0.0`** (3.2 is where `test.projects` arrived).
- **C15:** set build `mode` and `NODE_ENV=production` explicitly; no port/HMR isolation needed for Vitest.

## Spike 2: the D89 Shadow-DOM shell (K17)

**Verdict: viable; lock it in**, with one CSS fix and one design addition. Vite 8.3 + React 19.3 + Fluent UI v9.74 (Tabster 8.8), Playwright 1.64 with Chromium 156, Firefox 157 and WebKit 27.2, against a production build (and a `vite dev` smoke test: raw nesting in adopted sheets and `replaceSync` work).

**Held in all three engines:** user global CSS (incl. the `all: unset` reset, with `:host { all: initial !important }`) doesn't change shell elements; theme CSS doesn't reach showcases; slotted text gets UA defaults without a showcase stylesheet and the `[data-tss-showcase]` rules with one; the user's body font never reaches the shell; `var(--tss-*)` resolves in slotted content (not in body portals, as expected); the slot's `color-scheme` drives native controls; Fluent Dialog/Menu/Popover/Tooltip portal to `body`, are styled by Fluent, unclipped and on top; a non-portalled absolute dropdown isn't clipped; `position: fixed` stays fixed; exactly one `onClick` per click; plain Tab order shell ↔ showcase is natural; Fluent's focus trap and Escape work; `replaceSync` and the `<style>` fallback update the shell live; document-level `@font-face` works; theme switching restyles only the shell; hash/`:target` don't reach the shadow root while `scrollIntoView()` does; no console warnings; mounting 60 showcases takes 42–109 ms.

**Expected leaks, confirmed:** `html { font-size: 62.5% }` rescales the shell (rem); `html, body { display: flex; height: 100%; overflow: hidden }` breaks page scrolling. Both are already documented "must not"s.

**Problems found, with verified fixes:**

1. **`isolation: isolate` on `:host` is dead code:** `all: initial !important` resets `isolation` with `!important`, so a normal declaration loses (Vite's minifier even drops it). The host was not a stacking context, and the shell's z-indexed header painted over body portals without their own z-index. → `isolation: isolate !important`; **every declaration after `all: initial !important` in `:host` and `slot` must carry `!important`**.
2. **Focus libraries that walk the light DOM break keyboard order:** Fluent v9's Tabster (`autoRoot` on `body`) makes shell focusables unreachable around Fluent showcases. → `window.__tabsterShadowDOMAPI = getShadowDOMAPI()` (from `tabster`) before any `FluentProvider` mounts fixes it in all engines. The module can't hard-code library specifics, so modules offer a **showcase setup option**: a document-level module run before showcases mount (Storybook-preview style), next to `showcaseStylesheets`; C13 documents the Fluent/Tabster case.
3. **Slotted content stacks inside the host's stacking context:** a showcase's fixed/absolute element without a high z-index sits under the shell's z-indexed elements (sticky header). Body portals are fine (after fix 1). → Documented; keep the shell's `--tss-z-*` small.
4. **React synthetic events from showcases bubble to the shell's React ancestors** (normal portal behaviour). → The shell attaches no handlers on stage ancestors that react to showcase events, or stops synthetic propagation at the stage boundary.
5. **WebKit does honour `@font-face` inside shadow roots** (Chromium and Firefox don't). → The fonts stylesheet stays right; wording "ignored by Chromium and Firefox".
6. Document-level custom properties inherit into the shell (`all` doesn't reset them): a user's `--tss-*` on `:root` reaches any variable a theme doesn't define. Low risk (themes define the full set on `.tss-shell`); documented.
7. A user's `html { background }` competes with the module's canvas background on `<html>` (the D89 exception); documented, the module's rule is set inline and wins.

**Host note:** WebKit needed system libraries; the agent ran `npx playwright install-deps webkit` (passwordless `sudo -n`), which apt-installed packages (gtk4, gstreamer, libavif, …) on the user's WSL host. Not requested; reported to the user.

**Plan changes (to approve):**

- **C2 §8 rule 7 (contract):** `:host` gains `isolation: isolate !important`; every declaration after `all: initial !important` carries `!important`; the showcase setup option joins the showcase stylesheet; stacking of slotted content and synthetic-event bubbling stated.
- **C3 §1:** the same, plus the WebKit `@font-face` wording and the user `--tss-*`/`html` background notes.
- **C10:** the showcase setup option; no shell handlers reacting to bubbled showcase events. **C13:** focus libraries and shadow DOM (Fluent/Tabster worked example).
