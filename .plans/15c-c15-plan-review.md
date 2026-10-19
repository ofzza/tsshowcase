# Second review of the C15 plan (round 3): triage

A second clean-context agent reviewed C15 round 3 (2026-10-19, at the user's request). It did two things:

- checked that every `15b` finding and the decisions D138–D145 were applied correctly;
- looked for what the first review missed.

It checked claims against the Vite 8.3.4, Rolldown 1.2.14 and tinyglobby 0.2.17 sources in the spike's `node_modules`. It modified nothing and ran nothing.

**Verdict:** `15b` was applied correctly, except that S5's fix doesn't work as written (T1), and S10's escape lost its backslash (T7). The new findings are:

- a few Vite internals the plan assumed wrongly: plugin hook ordering (T2), CSS id matching (T3), and `NODE_ENV` being set during config loading (T4);
- one race in the live channel (T5);
- four decisions, one of which questions S1's complexity (T15).

18 findings (T1–T18). Claude's triage of each is in the last column. **Status (user, 2026-10-19): resolved.** Group A approved as triaged; T15 → D146, T16 → D147, T17 → D148 (with the user's afterthought: module options are root-only by default, and per-package ones are marked with `"x-tsshowcase-per-package": true`), T18: keep the `<style>` fallback (it makes testing easier and costs next to nothing). Applied in C15 round 4.

## A. Apply without a decision

| #   | Sev | Finding | Triage |
| --- | --- | ------- | ------ |
| T1  | high | **The S5 fix doesn't work.** The server ignores `import.meta.hot.invalidate()` unless the module accepts *itself*, and our showcases module accepts a dependency. Even from a self-accepting module, `invalidate()` propagates to importers rather than forcing a full reload. | Accept. In the accept callback: if the set of showcase files changed, swap the table; otherwise `location.reload()`. Fix the §6 wording. Add a §9 test with an edit to a showcase that isn't a refresh boundary. |
| T2  | medium | **Hook ordering:** `config` hooks are sorted by the hook's `order`, not by `enforce`. `environments.client` and `configEnvironment` hooks are applied after `config`, and the legacy `build()` and dev's `optimizeDeps` read the **client environment**. Rolldown `options`/`outputOptions` hooks can still change things. | Accept the facts. What to do depends on T15. In both variants: assert against the client environment's resolved values, in resolved form (`base` is `/` in serve, `publicDir: false` resolves to `''`); also protect `output.format: 'es'`; and fix the "post hooks run last" sentence. |
| T3  | medium | **`?tss-css` ids match Vite's CSS id regex** (`\.css(?:$\|\?)`), so `vite:css`/`vite:css-post` and dev import analysis would claim our JS module. The spike only exercised the themes virtual module. | Accept. Resolve to a `\0`-prefixed id that doesn't look like CSS (`\0tss-css:<abs>.js`), and keep `addWatchFile(<abs>)`. Build and dev test for a module CSS import. |
| T4  | medium | **Loading the showcase Vite config sets `NODE_ENV`.** The `runner` loader resolves a serve config, which sets `NODE_ENV=development` when it is unset. The §7 guard covers only `vite.build()`, so it would then "restore" `development` instead of deleting it. | Accept. The env guard wraps config loading **and** `build()`. Extend the §9 "unset `NODE_ENV` restored" test with a `userConfigFile`. |
| T5  | medium | **Live messages before the app subscribes are lost.** `hello` is sent when the runtime is first evaluated. A reply or an update arriving before the module calls `onData`/`onTestUpdate` (an async mount) is dropped, and the runtime keeps no copy (D141). | Accept the reviewer's first option, which needs no state: the runtime sends `hello` for a stream when the **first listener subscribes** to it (and on reconnect), with the revision the module is showing (the embedded one at first). The server replies with newer state. D141 stays as is. |
| T6  | low | **The user config can still change dev settings:** `server.host: true` bypasses the non-loopback warning (which reads `ctx.host`), and `server.hmr: false` silently disables theme and showcase HMR. | Accept. The warning reads the resolved host. `server.hmr` is protected as "not `false`". |
| T7  | low | **Escaping typo:** the plan and `15b` say "escapes `<` as `<`" (the backslash was lost when the files were written). | Accept: `<`. |
| T8  | low | **`optimizeDeps.entries` are glob patterns.** Absolute paths with `[`, `(`, `{` or `*` match nothing and are silently dropped. Entries are fixed per `dev` session, so showcases added later cost one dependency-discovery reload. | Accept. Glob-escape the paths, and note in §3.2 that entries are fixed per session. |
| T9  | low | **`selectTheme` must be standalone.** It is serialised by Node-side code, so it can't live in the runtime module (which imports virtual modules), and it can't reference imported constants. | Accept. A separate import-free file; `THEME_STORAGE_KEY` is passed in as an argument. |
| T10 | low | **The S12 shape contract misses groups.** | Accept: every hierarchy node keeps `type` and `children`; units also keep `dir` and `tests`. |
| T11 | low | **`url()` details:** a `?#iefix`/`#id` suffix must not go into the import specifier, and string URLs in `image-set("x.png" 1x)` aren't `url()`s. | Accept. The suffix is appended to the imported URL. String URLs inside `image-set()` are rewritten like `url()`s. |
| T12 | low | **C11 guarantee missing:** if `onDataUpdate` carries a snapshot older than an already forwarded test update, the server snapshot regresses while `testsRevision` claims it's newer. | Accept. Record for C11: every `onDataUpdate` snapshot includes all test updates already forwarded. |
| T13 | low | **AGENTS resume state:** the tracker still says C15 "– 🔜 next" without its plan file, and the C2/C3/C6 rows don't mention the pending amendments. | Accept, applied now (not only at approval). |
| T14 | low | **`customLogger`/`clearScreen`** are config keys a user config can override, yet the plan lists them as "not part of the config". | Accept: protected (one output channel). |

## B. Needs a decision

| #   | Sev | Finding | Recommendation |
| --- | --- | ------- | -------------- |
| T15 | medium | **Is S1's three-level protection worth it?** Even with T2's fixes, plugins can still change things in Rolldown's `options`/`outputOptions` hooks or by mutating the config after `configResolved`. Full protection against plugins isn't achievable, and user plugins are trusted code anyway (C13 documents that config files are executed). | **Recommended (reviewer and Claude): assert, don't fight.** Merge the protected keys last, then run **one** `configResolved` check on the client environment's resolved values; it throws, naming the key. Add **output-level checks** in `buildSite`: a single output, `index.html` present, no `tss-assets/`, ES format, no absolute or machine-relative paths. This drops the re-applying post hook and the `configEnvironment` hook, which is less ceremony for the same practical guarantee: a misconfiguration fails loudly instead of being silently "fixed". *Alternative:* keep all three levels with T2's fixes (`config: { order: 'post' }` plus a post `configEnvironment` hook). |
| T16 | medium | **Which copy wins plugin de-duplication (revisits S16).** A user's `@vitejs/plugin-react` often carries options their code needs (emotion's `jsxImportSource`, babel plugins for styled-components). Dropping it breaks those showcases with no way for the user to fix it. The module's app ships prebuilt in `dist`, so the framework plugin mostly transforms the *user's* files. | **Recommended (reviewer and Claude): the user's copy wins**, with a warning. C10 verifies that the module's app works under a user-configured plugin-react. *Alternative:* the module's copy wins (as approved in S16), and C10 adds a `reactPluginOptions` pass-through option. |
| T17 | low-medium | **How the CLI knows the root-only keys (D145).** C6 now hard-codes four conventional names. A third-party module that uses one of them with a per-package meaning would be silently overridden. | **Recommended: modules declare it in their schema,** consistent with D116's `OPTION_FORMAT` marking: a C2 constant (e.g. `ROOT_ONLY_OPTION = 'x-tsshowcase-root-only'`), set to `true` on the property. C6 warns and drops marked keys found in package configs. `-base-react` marks its four keys, and `testRunner` (D136) follows the same mechanism. This needs a C2 amendment (B8 grows), a C6 change and a D145 update. *Alternative:* keep the fixed list and document the four names as reserved. |
| T18 | low | **The `<style>` fallback is unreachable in real browsers.** The theme CSS needs nesting (Safari 17.2, Chrome 120, Firefox 117), and every such browser supports constructable stylesheets (Safari 16.4, Firefox 101). | **Claude differs from the reviewer: keep it.** jsdom, used by our runtime tests and C10's smoke render (super-plan C10: "feature-detects constructable stylesheets (`<style>` fallback)"), doesn't implement `adoptedStyleSheets`, so the fallback is what tests run against. It is a few lines; the plan notes the browser floor. *Alternative (reviewer's):* drop it, and give tests a polyfill. |

## Checked and fine (reviewer)

- **`15b` correctly applied:**
  - S2: `attachDebugInfo: 'none'` exists in Rolldown 1.2.14, and Vite passes `experimental` through.
  - S3.
  - S4: explicit entries bypass the HTML scan; entries under `node_modules` are scanned.
  - S6, S7: the B8 and C3-A2 wording matches the original texts.
  - S8.
  - S9, apart from T5 and T12.
  - S11, apart from T6.
  - S13.
  - S14: the runner loader writes no `.vite-temp`.
  - S15: D145 is reflected in C6, the super-plan and the AGENTS rows.
  - S16 (mechanism; T16 is about which copy wins).
- **Vite and Rolldown mechanics:**
  - `reloadModule` on a `\0` virtual module works.
  - The legacy `build()` builds only the client environment.
  - A `hello` sent before the socket opens is buffered by Vite's client.
  - The theme files in `node_modules/.cache` aren't watched, so the explicit `reloadModule` is the only trigger, as designed.
- **Consistency with the other plans:**
  - C7 §7 and D137 are consistent with D143.
  - Super-plan R9, C10 and C11 are consistent with the plan.
  - Rule 4's selection order matches §5.
  - `BuildContext`/`DevContext` usage matches C2 §5.8; the doc assets match B6, D120 and D125.
