# Chunk plan C5: Markdown pipeline (CLI core)

> Status: **APPROVED** (round 3, 2026-10-19). Q1–Q6 answered (D118–D123); the fresh-eyes review `05a-c5-plan-review.md` triaged and applied (D124–D131). Approved together with C2 amendment B6, C3 amendment C3-A1, the C6 amendment, the C4 note and super-plan R7, all applied. B6's §7 item needed no C2 edit (the stale own-classes text was only in C3 §2.1, fixed by C3-A1). Decisions referenced as `D<n>` are in AGENTS.md §3. **Wording from plan 5 (D201, D202, `04b-c4-redesign-review.md`):** the issue's `file` is `path` (D202; §6's stale "`dir` is the doc's directory" is left for plan 6); §4.1's exact-case example differs in the base only, and §4.1 and the asset safety check say that a written path resolves in exact case, extension included (D201 governs classification).

## Goal

Turn every README and licence the showcase displays into a C2 `Doc`: raw `source`, themed `html`, `headings` and `assets` (D23, D71, D120). Markdown gets GFM, GitHub-compatible ids, sanitised raw HTML, class-based syntax highlighting (D39, D118), rewritten links (D73, D78, D121) and local media copied into the site (D120). Text docs become an escaped `<pre>`. The same parsing and link classification also give `check` its `docs/*` issues (D123).

C5 renders and checks documents and nothing else. It never decides **which** documents exist (C4 finds root and package docs, C9's hierarchy helper finds group READMEs, D78), never calls the module except through a `resolveRoute` callback it is given, and never writes files (C11 writes doc assets).

## 1. Where the code lives

Everything is internal to the CLI package, like C4's `scan/`:

```
packages/tsshowcase/src/markdown/
  link-targets/   repo path → link target: unit, package, group, home, repo file, missing (§4)
  links/          the rehype pass that rewrites href/src (§4, §5)
  ids/            GitHub-compatible heading ids, footnote ids, author anchors, Doc.headings (§3)
  highlight/      code-block highlighting onto tss-token-* classes (§6)
  sanitise/       the raw-HTML schema (§2.2)
  alerts/         GitHub alerts → tss-alert (§2.3)
  text-doc/       Doc.format 'text' → <pre class="tss-text-doc"> (§7)
  assets/         local media → doc assets: safety checks, hashing, tss-assets/ paths (§4.6)
  render-docs/    renderDocs() and checkDocs(): tie it together (§8)
```

New runtime dependencies of the CLI package (versions fixed at implementation): `unified`, `remark-parse`, `remark-gfm`, `remark-gemoji`, `remark-rehype`, `rehype-raw`, `rehype-sanitize`, `rehype-stringify`, `github-slugger`, `hast-util-to-string`, `unist-util-visit`, and `@wooorm/starry-night` (D118).

## 2. The markdown pipeline

```
remark-parse → remark-gfm → remark-gemoji → remark-rehype (allowDangerousHtml, clobberPrefix: '')
  → rehype-raw → rehype-sanitize (§2.2) → footnote ids (§3.3) → alerts (§2.3) → ids (§3) → links (§4)
  → highlight (§6) → generated classes (§2.1) → rehype-stringify
```

Sanitising comes right after raw HTML is parsed, so every later pass works on trusted structure and its own additions (ids, classes, rewritten URLs) are never stripped.

**GFM features:** tables, task lists (disabled checkboxes), strikethrough, autolinks, footnotes, GitHub emoji shortcodes (`:tada:` → Unicode, no images). **Not supported in v1** (rendered as plain text or as an ordinary code block, documented): math (`$…$`), Mermaid diagrams, GitHub's `@mention`/`#123` autolinks, front matter.

Output is **deterministic** (same inputs, same HTML and `assets`), for reproducible builds.

### 2.1 Classes and attributes in the output (C3 §4)

The module puts README HTML inside `.tss-content` (C2 rule 6), where the theme styles bare elements (D92), and a content element shares one rule with its class (`.tss-paragraph, .tss-content p`). So C5 adds a class **only where a bare-element selector can't tell elements apart**:

| Element | Class / attribute |
| ------- | ----------------- |
| `<pre>` of a fenced code block | `tss-code-block`, plus `data-language` (`tssData.language`, C3-A1) when the language is known (§6) |
| highlighted token `<span>` | `tss-token-<token>` (§6) |
| the permalink inside each heading | `tss-heading-anchor` |
| GitHub alert | `tss-alert` with `data-alert` (`tssData.alert`), its title `tss-alert-title` (§2.3) |
| the footnotes section's hidden "Footnotes" label | `tss-visually-hidden` (remark-rehype's `sr-only`, mapped) |
| text doc `<pre>` | `tss-text-doc` (§7) |

**Classes the libraries generate are replaced or stripped** in the last pass: `sr-only` → `tss-visually-hidden`; `footnotes`, `data-footnote-backref`, `contains-task-list`, `task-list-item` and `language-*` (after the highlighter has read it) are removed. remark-rehype's `data-footnotes`, `data-footnote-ref` and `data-footnote-backref` attributes are removed too. A footnotes section stays recognisable as `section` with the `tss-visually-hidden` label (themes style `.tss-content section`, `.tss-content sup`).

Everything else is bare HTML (`p`, `a`, `ul`, `table`, `blockquote`, `hr`, inline `code`, `sup`, `section`, …). No `style` attribute is ever emitted, and no class outside the catalogue (both asserted on the whole test corpus).

### 2.2 Raw HTML (H23, D119)

READMEs written for GitHub use raw HTML (`<p align="center">`, `<details>`, `<kbd>`, `<img width>`, `<br>`). C5 **passes it through, sanitised with GitHub's rules**: `hast-util-sanitize`'s default schema (modelled on GitHub's, which already allows task-list checkboxes), changed as follows:

- **`class`:** dropped from author markup (GitHub drops it too), so READMEs stay portable and only C5 adds catalogue classes (D126). The exception is `language-*` on `code`, which the default schema keeps and the highlighter needs (§6; removed after it, §2.1). `style` is dropped (default).
- **Author `id`/`name`:** keep the default `user-content-` prefix, so they can't clash with C5's ids; they are valid anchor targets (§3.4, D128).
- **Media:** `picture`, `source` (`src`, `srcset`, `media`, `type`, `width`, `height`), `video` (`src`, `poster`, `controls`, `loop`, `muted`, `playsinline`, `width`, `height`) and `audio` (`src`, `controls`, `loop`, `muted`) are allowed, for doc assets (§4.6). `autoplay` is not.
- **URL protocols:** the default (`http`, `https` for media; plus `mailto` etc. for links). `data:` and `javascript:` URLs are removed, in markdown and raw HTML alike.

Scripts, event handlers, `<iframe>`, `<style>` and forms are removed. The rendered HTML is safe to insert verbatim (rule 6). Presentational attributes the default schema keeps (`align`, `width`, `height`, `border`) are **user content**, like showcase code: outside D46, which binds tsshowcase's own modules and themes only.

### 2.3 GitHub alerts (D122)

A **top-level** blockquote whose first line is exactly one of `[!NOTE]`, `[!TIP]`, `[!IMPORTANT]`, `[!WARNING]`, `[!CAUTION]` (case-insensitive, nothing else on that line) becomes:

```html
<div class="tss-alert" data-alert="note" role="note"><p class="tss-alert-title">Note</p>…</div>
```

Anything else (a marker with trailing text, a nested blockquote, an unknown kind) stays an ordinary blockquote. Classes, `tssData.alert` and alert tokens come with C3-A1.

## 3. Ids

The id set of a doc is everything a `#fragment` may point at: heading ids, footnote ids and author anchors. Ids are unique **per doc** (H36).

### 3.1 Heading ids

- **They follow GitHub** (`github-slugger`, a fresh slugger per doc): lowercase, punctuation dropped, spaces → `-`, duplicates suffixed `-1`, `-2`, …. So `#usage` links written for GitHub keep working, inside the doc and across docs.
- Every `h1`–`h6`, including raw-HTML headings, gets an id and a permalink `<a class="tss-heading-anchor" href="<own route with heading>" aria-label="Link to section: <text>">#</a>` appended. Docs without a route of their own (§4.3) get the id but no permalink.
- **Raw-HTML headings with an author `id`** (R5): the slug id replaces it on the heading itself; the author id (already `user-content-…`) moves to an empty `<a id>` at the start of the heading, so links to it keep working (§3.4).
- **Exception:** the footnotes section's hidden label (remark-rehype's `h2#footnote-label`) gets no slug, no permalink and no `Doc.headings` entry; it keeps its `footnote-label` id.

### 3.2 `Doc.headings`

`{ depth, text, id }` in document order, `text` the heading's plain text (`hast-util-to-string`, without the permalink). Empty for text docs.

### 3.3 Footnote ids (D127)

remark-rehype builds footnote ids from the **label**, not the visible number: `[^note]` → `fn-note`, references `fnref-note`, repeated references `fnref-note-2`, … (label lowercased and URI-normalised). C5 keeps them **bare**: `remarkRehype({ clobberPrefix: '' })`, then, because the sanitiser prefixes every `id`, `name`, `aria-describedby` and `aria-labelledby` with `user-content-`, a pass right after sanitising strips that prefix again from the footnote ids, their `href`s and the `aria-describedby="footnote-label"` references, and from nothing else. Footnote links are rewritten like in-page anchors (§4.2). Tested with labels like `[^Note 1]`.

### 3.4 Author anchors (D128)

Author `id`/`name` attributes (`<a name="x">`, `<div id="x">`) are valid targets, as on GitHub, where `#x` reaches `user-content-x`. In the HTML they stay `user-content-x`; a link `#x` that matches one passes the **real element id** `user-content-x` as `resolveRoute`'s `heading`. So `heading` is always an element id present in the doc's HTML, and C10's router just looks it up (C2 amendment B6). When `#x` matches both a heading id and an author anchor, the heading wins (GitHub's behaviour).

## 4. Links (D73, D78, D121, M6)

### 4.1 Classifying a link

Applies to `a[href]`. Media URLs follow the same classification but end up as doc assets (§4.6).

1. **External:** a scheme (`https:`, `mailto:`, …) or protocol-relative `//` → untouched (`rel="noopener noreferrer"` added; nothing opens in a new tab).
2. **In-page anchor** (`#slug`) → §4.2.
3. **Repo path:** resolved against the doc's directory; a leading `/` is the repo root (as on GitHub). Percent-decoded, query and fragment split off. A path normalising outside the repo root → **missing**. (Known false positive, documented: a repo scanned from a monorepo subdirectory (`vcs.directory`) whose READMEs link `../../other/README.md`, which GitHub would resolve.)
4. The path is looked up in this order (`link-targets/`):

| Repo path is … | Target |
| -------------- | ------ |
| the root README (existing) or the repo root | home (`single` layout: home wins over package `'.'`, the package level is collapsed, C2 §4.3) |
| a package's README (existing) or directory | that package, if `analyze` kept it (D72); otherwise a repo file |
| a unit's README (existing) or directory (filtered analysis, D106) | that unit |
| a group's README (existing) or directory | that group if it has a route (D78); otherwise a repo file |
| an existing, non-ignored file or directory | a repo file → the VCS host (§4.4) |
| ignored or nonexistent | **missing** (§4.5) |

A README path that doesn't exist (a unit whose module expects a `README.md` that isn't there) is missing, as on GitHub; the directory rows still match the directory itself.

Units dropped for being outside `scope` (D106) are not in the filtered analysis, so links to them fall through to "repo file" (they exist, just not showcased; M6).

**Exact case (D130):** matching is case-sensitive, as on GitHub, on every filesystem. Existence uses `RepoFiles.exists` (D108: ignored = nonexistent), and on case-insensitive filesystems (macOS, Windows) the exact spelling of each segment is confirmed with a `list` of its parent; both are tracked reads in `dev`. So `./Readme.md` is missing everywhere when the file is `README.md` (a written path resolves in exact case, extension included; D201 governs classifying the resolved file, not resolving a written path), and the same repo gives the same links, assets and issues on every machine.

### 4.2 Anchors

- `#slug` in a doc with a route → `resolveRoute(ownTarget, slugOrElementId)`; `target#slug` (another README) → `resolveRoute(target, slugOrElementId)` (§3.4 for author anchors).
- **Validation never depends on routes (D124):** the fragment is checked against the **target doc's id set** (§3) whenever the target is a markdown doc of the set, whether that doc has a page or not. An unknown fragment is a `docs/broken-anchor` issue (§8.1); the link then goes to the target's route without a heading (or keeps its fragment on a VCS URL). A target without a doc (a unit directory with no README) has no id set: its route, without a heading, no issue.
- Fragments on links to other repo files are kept on the VCS URL (`…/blob/HEAD/docs/guide.md#setup`) and not validated (only the docs of the set are parsed).

This needs every target doc's ids before the linking doc is processed, so both `renderDocs` and `checkDocs` work in two passes over the whole set (§8): parse and collect ids, then classify (and render).

### 4.3 Docs without a route of their own

A group README whose group has no route (D78), and licences, have no in-app page of their own: in-page anchors there become plain text (the `<a>` is unwrapped, its children kept), and headings get ids but no permalink. Their fragments are still validated (D124): a typo is a typo whether or not the doc has a page.

### 4.4 Repo files → VCS host (D24, D110)

`VcsInfo.fileUrlTemplate` with `{path}` filled: the repo-relative path, URL-encoded **per segment** (`encodeURIComponent`, `/` kept); query and fragment from the link appended. Directories use the same template (C4 §9: GitHub and GitLab redirect `blob` to `tree`, Bitbucket's `src` serves both).

**No VCS info** (no `VcsInfo`, or host `other` without a template): the link is unwrapped to its text and `RenderedDocs.noVcsLinks` is set; C11 prints its one-time warning from that (D110). Not an issue: nothing is broken, the setting is just missing.

### 4.5 Missing targets (D121)

A link to an ignored or nonexistent path (D103: ignored = nonexistent) is **unwrapped** to its text with a `docs/broken-link` issue (§8.1). A VCS URL would 404 for a gitignored file, and a relative URL would 404 in the deployed site.

### 4.6 Images and other media: doc assets (L1, D120, D125, D129)

**Why copy:** the showcase must be portable and the source repo may be private, so the site can't point at the VCS host for images (a `blob` URL isn't even an image; a `raw` URL fails for private repos, shows the default branch instead of the working copy, and needs the host to be reachable). And there is no way to know whether a file is or will be committed, or on which branch, so **the showcase is built from the local state**, as it already is for READMEs, tests and showcases (the user's reasoning, D129; the "real path not ignored" condition was Claude's addition, approved with the plan).

**Which URLs:** `img[src]`, `img[srcset]`, `picture > source[srcset]`, `video[src|poster]`, `video > source[src]`, `audio[src]`, `audio > source[src]`, in markdown (`![alt](url)`) and raw HTML alike. Every `srcset` candidate is handled separately, keeping its `1x`/`640w` descriptor. A link (`<a href="./docs/guide.pdf">`) is not media: it still goes to the VCS host (§4.4).

**The contract (C2 §4, amendment B6):**

```ts
export interface Doc {
  // … as before …
  readonly assets: readonly DocAsset[]; // local media this doc's HTML refers to; empty for text docs
}

export interface DocAsset {
  readonly source: RepoPath; // the repo file, e.g. 'packages/ui/src/Button/docs/states.png'
  readonly path: string; // where it lives in the site, relative to the site root, e.g. 'tss-assets/3f9a1c0b7d2e4f68/states.png'
}
```

**The rewrite, step by step** (in the links pass):

1. Classify the URL like a link: external (`http:`/`https:`) → untouched (`data:` was already removed by the sanitiser, §2.2); a repo path → resolved against the doc's directory, percent-decoded, query and fragment dropped.
2. **Safety (D129).** The path must name an existing, non-ignored **file** with its exact case, extension included (D108, D130; D201 doesn't apply to resolving written paths), and:
   - its **real path** (every symlink resolved, in the file itself and in every parent directory) must lie inside the **real path of the repo root**; a symlink anywhere along the way that leads outside the repo disqualifies it;
   - the real path must itself be non-ignored, so an in-repo symlink can't expose a gitignored file (`docs/x.png -> ../.env`).

   Otherwise the media is dropped, with a `docs/broken-link` issue (§8.1): the whole element for `src`/`poster` (an image's `alt` text kept as text), only the bad candidate for `srcset` (the element is removed when no candidate is left).
3. Read the file's bytes through C4's tracked file layer (binary reads next to `read`, C4 note; tracked, so `dev` sees a changed image) and hash them: `sha256`, first 16 hex characters.
4. The site path is **`tss-assets/<hash>/<file name>`**. The hash makes it stable across machines and builds, changes it whenever the content changes (no stale browser caches), and deduplicates a file referenced from several docs. The original file name keeps the extension (correct MIME type) and readable URLs. No repo directory structure appears in the site. `tss-assets/` (`DOC_ASSETS_DIR`) is reserved for doc assets (no underscore prefix: GitHub Pages' Jekyll would drop it).
5. The attribute becomes that path, percent-encoded per segment: `src="tss-assets/3f9a1c0b7d2e4f68/states.png"`. It is **relative to the site root**: under rule 1, `index.html` at the site root is the document every page runs in (hash routing). A module that inserts doc HTML into a document elsewhere sets `<base>` or rewrites the URLs (B6, rule 6). The doc lists `{ source, path }` in `Doc.assets`, once per path.

**Example.** `packages/ui/src/Button/README.md` contains:

```md
![Button states](./docs/states.png)

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="./docs/dark.png 1x, ./docs/dark@2x.png 2x" />
  <img alt="Sizes" src="./docs/light.png" />
</picture>
```

It renders as:

```html
<p><img src="tss-assets/3f9a1c0b7d2e4f68/states.png" alt="Button states"></p>
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="tss-assets/91d0e6a2b4c87f13/dark.png 1x, tss-assets/0c55aa19e3b27d40/dark@2x.png 2x">
  <img alt="Sizes" src="tss-assets/7be24f0d1a9c3e65/light.png">
</picture>
```

with `Doc.assets`:

```ts
[
  { source: 'packages/ui/src/Button/docs/states.png', path: 'tss-assets/3f9a1c0b7d2e4f68/states.png' },
  { source: 'packages/ui/src/Button/docs/dark.png', path: 'tss-assets/91d0e6a2b4c87f13/dark.png' },
  { source: 'packages/ui/src/Button/docs/dark@2x.png', path: 'tss-assets/0c55aa19e3b27d40/dark@2x.png' },
  { source: 'packages/ui/src/Button/docs/light.png', path: 'tss-assets/7be24f0d1a9c3e65/light.png' },
]
```

**Who writes and serves the files (D120, D125):**

- **`build`: the CLI.** C11 collects every `Doc.assets` entry from the data (deduplicated by `path`) and writes the files next to `BuildResult.files` (D64). Modules do nothing. A module file under `tss-assets/` is an error (reserved prefix, rule 1).
- **`dev`: the CLI copies, the module serves a directory.** C11 keeps a doc-assets directory in the CLI cache (`node_modules/.cache/tsshowcase/doc-assets/`, laid out like the site: `tss-assets/<hash>/<name>`), rebuilt from every data update so paths no longer referenced disappear, and passes it as `DevContext.docAssetsDir` (absolute). The module serves that directory statically at `tss-assets/` and nothing else there (`-base-vite`, C15). All path-safety logic stays in the CLI.

**Limits:** SVGs are copied as files and shown through `<img>`, which never runs their scripts. No size limit in v1. A copied file can still be uncommitted, later removed or replaced, exactly like READMEs, tests and showcases (D129).

**Not supported in v1 (documented):** GitHub's `#gh-dark-mode-only` / `#gh-light-mode-only` image fragments (both images show; `<picture>` with `prefers-color-scheme` works, though it follows the OS, not the selected theme).

**External image URLs** (badges) are left alone: they are the README author's own content, like requests made by showcase code; C2 rule 2 gains that exception explicitly (B6).

## 5. The route callback

C5 never sees the module. `renderDocs` receives `resolveRoute: (target: RouteTarget, heading?: string) => string | undefined`, bound by C11 to the module and the current analysis (C2 §5.6: synchronous, pure). A module error (a throw, or `undefined` for home/package/unit targets) propagates as a module error naming the module; C11 formats it. **`checkDocs` never calls it** (D124).

## 6. Syntax highlighting (D39, K22, D118, D131)

**`@wooorm/starry-night`** uses the TextMate grammars GitHub uses (code looks as it does on GitHub, TSX and type syntax included), outputs hast and emits **classes only** (`pl-k`, `pl-s`, `pl-c1`, …), never styles. It runs at build time in Node; only classes reach the browser. The `common` grammar set ships (no `all`: install size); grammars load once per process.

- **Mapping onto the `tss-token-*` classes** (draft; finalised against starry-night's shipped CSS at implementation, and published in the CLI README and the catalogue docs):

  | starry-night | token |
  | ------------ | ----- |
  | `pl-c` | `comment` |
  | `pl-k` | `keyword` (GitHub's `pl-k` also covers operators and storage) |
  | `pl-s`, `pl-pds` | `string` |
  | `pl-c1` | `constant` (GitHub's `pl-c1` also covers numbers and support names) |
  | `pl-en` | `function` (also class and type names) |
  | `pl-v` | `variable` |
  | `pl-ent` | `tag` |
  | `pl-e` | `attribute` |
  | `pl-sr`, `pl-cce`, `pl-sre`, `pl-sra` | `regexp` |
  | `pl-mi1` | `inserted` |
  | `pl-md` | `deleted` |
  | everything else (`pl-smi`, `pl-s1`, `pl-ii`, markup classes, …) | dropped (renders in `--tss-code-text`) |

- **Coverage (D131):** with starry-night, `number`, `type`, `operator`, `punctuation` and `property` are rarely or never emitted distinctly, because GitHub's classes don't separate them. They stay in C3's 16 tokens (other highlighters and modules may use them), and the docs say so, so theme authors know which variables are live. GitHub's contextual styling (`.pl-s .pl-v`) is flattened.
- The "every emitted class is mapped or deliberately dropped" test derives the class list from starry-night's shipped CSS (no API exports it).
- **Languages:** the code's `language-*` class (from the fence info string, kept by the sanitiser) → `flagToScope` over the registered grammars (`ts`, `tsx`, `js`, `jsx`, `json`, `css`, `html`, `md`, `sh`/`bash`/`shell`, `yaml`, `diff`, …). Unknown or missing language → unhighlighted, no issue. `data-language` on the `<pre>` carries the normalised flag; `language-*` is then removed (§2.1).

## 7. Text docs (D71)

`Doc.format: 'text'`: `html` is `<pre class="tss-text-doc">` with the source HTML-escaped, no linkification; `headings: []`; `assets: []`. A **markdown licence** (`LICENSE.md`) is a markdown doc like any other: rendered, checked, without an owner (§4.3).

## 8. API

```ts
export interface DocInput {
  readonly path: RepoPath;
  readonly format: 'markdown' | 'text';
  readonly source: string;
  readonly owner?: RouteTarget; // the page this doc is shown on (home, a package, a routed group, a unit); unset for licences and route-less group READMEs. Rendering only.
}

export interface LinkTargets {
  readonly layout: 'single' | 'workspace';
  readonly rootReadme?: RepoPath;
  readonly packages: readonly { readonly dir: RepoPath; readonly readme?: RepoPath; readonly shown: boolean }[]; // shown: kept by analyze (D72)
  readonly units: readonly { readonly dir: RepoPath; readonly package: RepoPath; readonly readme?: RepoPath }[]; // the filtered analysis (D106)
  readonly groups: readonly { readonly dir: RepoPath; readonly package: RepoPath; readonly readme?: RepoPath; readonly routed?: boolean }[]; // routed: rendering only
}

export interface CheckDocsOptions {
  readonly docs: readonly DocInput[];
  readonly targets: LinkTargets;
  readonly files: TrackedRepoFiles; // C4's file layer: existence, case, real paths, binary reads (tracked in dev)
  readonly isScanned: (dir: RepoPath) => boolean; // C4's RepoScan.isScanned, for Issue.dir
}

export interface RenderDocsOptions extends Omit<CheckDocsOptions, 'isScanned'> {
  readonly vcs?: VcsInfo;
  readonly resolveRoute: (target: RouteTarget, heading?: string) => string | undefined;
}

export interface RenderedDocs {
  readonly docs: ReadonlyMap<RepoPath, Doc>;
  readonly noVcsLinks: boolean; // a repo-file link was unwrapped for lack of VCS info (C11's one-time warning, D110)
}

export function renderDocs(options: RenderDocsOptions): Promise<RenderedDocs>;

/** D123, D124: the same parsing, ids and classification, without routes, hashing or HTML; for `check`. */
export function checkDocs(options: CheckDocsOptions): Promise<readonly Issue[]>;
```

- **Whole set at once:** two passes (parse and collect id sets, then classify and render), because cross-doc anchors need the target's ids (§4.2). In `dev`, C11 re-renders the whole set when any doc, any asset source, the structure or the analysis changes; docs are few and rendering is fast, so no per-doc invalidation in v1.
- **One classifier (D124):** `renderDocs` and `checkDocs` share parsing, ids, the §4.1 classification and the §4.6 safety checks; whether something is broken never depends on routes. So `check` and the built site always agree on what is broken; only the rendering of correct links depends on routes.
- C11 builds `LinkTargets` from C9's hierarchy (§8.2) plus group routes and turns `RenderedDocs` into `RepoNode.readme`/`licence`, `PackageNode.readme`/`licence`, `GroupNode.readme`, `UnitNode.readme`.

### 8.1 Doc issues in `check` (D123)

`check` reports broken links and anchors as **warnings** (CI sees them; they fail it only when the user raises the severity through `rules`, D70):

| Rule | Default | When |
| ---- | ------- | ---- |
| `docs/broken-link` | warning | a link or media URL points at an ignored or nonexistent path, a wrongly cased one, a path outside the repo, or media failing the safety checks (§4.1, §4.5, §4.6) |
| `docs/broken-anchor` | warning | a fragment names no id in the target doc (§4.2) |

- `path` (D202, was `file`) is the doc, `line` the link's line (best-effort inside raw HTML, whose positions come from parse5), `message` names the URL as written, `hint` says what was looked for (e.g. "no heading `#instalation` in `README.md`; did you mean `#installation`?" when a near match exists). `dir` is the doc's directory when it is scanned; otherwise unset, so the issue shows at package or repo level (like N17).
- Which docs: the root README, every package README (hidden packages too, D72: their READMEs exist on the host), every unit README of the filtered analysis (D106), every group README (D78) and every markdown licence (§7).
- `docs/*` joins `scan/*` and `module/error` as CLI-known rule ids, so `rules` can override them (C6 amendment).
- **`build` and `dev` don't report them a second time:** they run `check` first (D76). In `dev`, C11 re-runs `checkDocs` when it re-renders.
- Missing VCS info is not an issue (nothing is broken, the setting is just missing); `build` warns once (C11, D110).

### 8.2 The hierarchy helper moves to C9 (super-plan R7)

`checkDocs` needs the group directories and their READMEs, which C11 was to derive while building the hierarchy. So **building the hierarchy from flat units + `unitRoot` (groups, nesting, group READMEs, D78)** becomes a shared helper owned by **C9** (next to the analysis validation and the `scope` drop it follows), reused by C11, which adds routes. In the C2 CLI-duties table, that row moves from C11 to C9.

**Implementation order:** C9 is implemented before C5 (D29), unchanged. C9 reserves a docs step in `check` and the place of `docs/*` issues in the report; **C5's implementation wires `checkDocs` into that step** and adds a `check` integration test on a fixture with broken links and anchors. Before C5 lands, `check` has no `docs/*` issues.

## 9. Tests

Unit tests per internal directory, on in-memory inputs (a fake file layer that can simulate case-insensitive filesystems and symlinks); one fixture for the `check` integration test (§8.2):

- pipeline: a markdown corpus (every GFM feature, emoji, raw HTML, nested lists, tables, footnotes, task lists) rendered to HTML snapshots; determinism (two runs equal); **no `style` attribute** and **no class outside `tssClass`** anywhere in the corpus output;
- sanitising: an XSS corpus (`<script>`, `onerror`, `javascript:` and `data:` URLs, `<iframe>`, `<style>`, SVG with script, `autoplay`, author `class`/`style`/`id`); `language-*` survives to the highlighter;
- ids: GitHub parity on a table of tricky headings (punctuation, emoji, duplicates, non-Latin, inline code); raw-HTML headings with and without an author id; per-doc uniqueness; footnote ids from labels (`[^note]`, `[^Note 1]`, repeated references) without any `user-content-` prefix, `aria-describedby` intact; the footnote label not in `Doc.headings`; author anchors (`name`, `id`) as targets; heading wins over an author anchor;
- links: every row of §4.1 incl. the single-layout collapse, hidden packages (D72), dropped units (D106), routed and route-less groups, missing READMEs, root-relative and `..` paths, out-of-root paths, percent-encoding, query and fragment, exact case on a simulated case-insensitive filesystem, ignored and missing targets, no VCS info;
- VCS: `HEAD` links for GitHub, GitLab and Bitbucket (`src/HEAD/…`), with and without `vcs.directory`, a user template, segment encoding (spaces, `#`, Unicode) (C4 "Recorded for later chunks");
- anchors: in-page, cross-doc, unknown fragment, target without a doc, route-less own doc (validated, unwrapped), fragment kept on VCS URLs;
- assets: the §4.6 example verbatim; dedup across docs; same content under two names; a changed file gets a new path; missing, ignored, wrongly cased and out-of-repo media; a symlinked file and a symlinked parent directory leading outside the repo; an in-repo symlink to an ignored file; a directory as `src`; SVG; `srcset` with one bad candidate; percent-encoded file names; text docs have none;
- `checkDocs`: every broken case yields exactly the issue `renderDocs` acted on (same classifier), regardless of routes; `dir` set or unset by scan state; near-match hints; markdown licences;
- highlighting: the mapping covers every class in starry-night's CSS, `diff` → inserted/deleted, unknown language, TSX sample snapshot;
- alerts: all five kinds, lower-case markers, a marker with trailing text, a nested blockquote, an ordinary blockquote;
- text docs: escaping (`<`, `&`), no headings, no assets.

## 10. Documentation

- CLI README, "How READMEs are rendered": supported markdown (GFM, emoji, alerts), what isn't (math, Mermaid, mentions, `#gh-dark-mode-only`), raw HTML and what the sanitiser removes (including author classes), how each kind of link is rewritten (the §4.1 table in user terms), exact case, local images copied to `tss-assets/` from the local state and the safety rules, external images, highlighted languages and the token mapping with its coverage (D131), heading ids matching GitHub, the `HEAD` links (D110), the `vcs.directory` false positive; the `docs/*` rules in the rule list.
- The `catalogue/` README (C3's generator): which classes and attributes C5 emits (§2.1), that bare elements inside `.tss-content` cover the rest, and which syntax tokens starry-night leaves unused.
- `CONTRIBUTE.md` glossary: doc, link target, repo file, doc asset, id set.

## Acceptance criteria

- The corpus renders to the reviewed snapshots; output contains no `style` attribute and no class outside the catalogue.
- Every row of the §4.1 table, every anchor case and every host's `HEAD` link is covered by a test.
- The XSS corpus renders without any script, handler, `data:` or `javascript:` URL.
- The §4.6 example produces exactly the documented HTML and `Doc.assets` (with the real hashes of the test files); no symlink case publishes a file outside the repo or an ignored file.
- `checkDocs` and `renderDocs` agree on every broken case.
- `npm run ci` passes.

## Out of scope

Finding root and package docs (C4); the hierarchy helper and the docs step's place in `check` (C9); group routes, writing doc assets in `build`, the `dev` doc-assets directory, printing the `no-vcs` warning (C11); serving `docAssetsDir` in `dev` (C15); scrolling to ids in the app (C10); the styles for alerts and README elements (C3-A1).

## Recorded for later chunks

- **C9:** the shared hierarchy helper (groups, nesting, group READMEs, D78), reused by C11 (§8.2); a reserved docs step in `check` (wired by C5's implementation); `docs/*` issues in the report like any other.
- **C10:** the router scrolls to any element id in the doc's container (headings, footnotes, author anchors `user-content-…`, B6, D128); a module showing several docs on one page must scope or prefix their ids, which are unique per doc only (H36, R37).
- **C11:** builds `LinkTargets` (with `shown` from the analysis, `routed` from group routes) on top of C9's hierarchy; calls `renderDocs` once per data assembly and on every `dev` change to docs, asset sources, structure or analysis, re-running `checkDocs` then; the one-time `no-vcs` warning from `RenderedDocs.noVcsLinks` (D110); **writes every `Doc.assets` file** (deduplicated by `path`) next to `BuildResult.files` and rejects module files under `tss-assets/`; the "no absolute build-machine paths" output check skips doc assets (user content, binary); keeps the `dev` doc-assets directory in the CLI cache, rebuilt per data update, passed as `DevContext.docAssetsDir` (D125).
- **C12:** a README image in `build` and in `dev` (updated when the image changes), footnote and cross-doc anchor scrolling, author anchors, themed alerts.
- **C13:** README authoring notes (what renders like GitHub, what doesn't; local images and their safety rules; exact case; the `docs/*` rules); ids unique per doc for module authors (R37).
- **C15:** in `dev`, serves `DevContext.docAssetsDir` statically at `tss-assets/` and nothing else under that prefix.
- **v1 cut line (K21, D98):** candidates: alerts, `video`/`audio`, near-match hints (R38).

## C2 amendment B6 (approved with this plan, applied)

- **§4 `Doc`:** `assets: readonly DocAsset[]` (empty for text docs) and `DocAsset { source: RepoPath; path: string }` (§4.6); the `Doc.html` comment adds "local media rewritten to doc assets".
- **§5.6:** `heading` is an element id present in the doc's HTML that C5 produced or kept: a heading id from `Doc.headings`, a footnote id, or an author anchor (`user-content-…`) (§3, D128). The "Link mapping for C5 (D73)" bullet adds: local media → doc assets (D120); ignored or nonexistent targets → unwrapped text (D121); fragments are validated against the target doc's ids (D124).
- **§5.8 `DevContext`:** `docAssetsDir: string` (absolute; the CLI-maintained directory laid out like the site's `tss-assets/`), served by the module at `tss-assets/` (D125). `BuildResult.files` comment: "the site, except doc assets, which the CLI adds".
- **§8 rule 1:** "the returned files, plus the doc assets the CLI adds, form a self-contained static site"; the prefix `tss-assets/` is reserved for doc assets, written by the CLI; a module returns no file under it. Inclusion constant `DOC_ASSETS_DIR = 'tss-assets'`.
- **§8 rule 2:** "… unless the user's own showcase code or README content (external images, badges) makes them".
- **§8 rule 6:** URLs in doc HTML (doc assets) are relative to the site root; a module inserting doc HTML into a document elsewhere sets `<base>` or rewrites them.
- **§7 (C2 text on own classes, R20):** a theme's own classes are useful in shell-mode showcases only; author classes in README raw HTML are stripped (D126).
- **§5.9 / §9:** `check` also reports `docs/*` issues (§8.1).

## C3 amendment C3-A1 (approved with this plan, applied)

- **Alerts (D122):** classes `tss-alert`, `tss-alert-title`; `tssData.alert` (`data-alert`: `note`, `tip`, `important`, `warning`, `caution`); semantic tokens `--tss-alert-<kind>` and `--tss-alert-<kind>-surface`, defaulting to existing semantic tokens (note and important → accent, tip → `passed`, warning → a warning pick, caution → `failed`; final picks at implementation); default theme styles; contrast reviewed (D85).
- **`tssData.language`** (`data-language`, open values): the language of a code block (§2.1).
- **README elements in the default `catalogue.css`** (inside `.tss-content`): `img`, `video`, `picture` (`max-width: 100%`), `kbd`, `details`/`summary`, task-list checkboxes, footnote `sup` and `section`, using existing tokens.
- **Text fix (R20, D126):** §2.1 rule 3 and D86: own classes are useful in shell-mode showcases only (README raw HTML loses author classes).
- **§3.3 / catalogue docs (D131):** note which syntax tokens the CLI's highlighter leaves unused.

## C6 amendment (approved with this plan, applied)

- **§12 rule ids (M15):** the CLI-known rule ids gain `docs/broken-link` and `docs/broken-anchor`, and `docs/` becomes a CLI-owned namespace (unknown ids in it warn, like `scan/` and `module/`).

## C4 note (approved with this plan, applied)

- **§7 `files/`:** binary reads (bytes) next to `read`, and real-path resolution, both tracked as reads and covered by C4's tests; a case-exact existence check (parent `list`) available to C5 (§4.1, §4.6).

## Super-plan R7 (approved with this plan, applied)

- **C5:** `check` issues `docs/broken-link`, `docs/broken-anchor` (D123, D124); doc assets copied by the CLI (L1, D120, D125, D129); alerts (D122); starry-night (D118); depends on C2, C3, **C4** (the file layer).
- **C2 CLI duties table:** "Building the hierarchy from flat units + `unitRoot` (groups, nesting, group READMEs)" moves from C11 to **C9 (shared helper, reused by C11)**; new rows "writing doc assets into the output" → C11 and "the `dev` doc-assets directory" → C11.
- **C9:** the hierarchy helper and group README lookup (moved from C11's "Group READMEs (D78)" bullet); a reserved docs step in `check`; depends on C4, C5, C6, C8. **C11:** writes doc assets, maintains the `dev` doc-assets directory. **C15:** serves `docAssetsDir`. **C12:** the C5 E2E items.
- **Dependency edges:** C5 → C2, C3, C4; C9 → C4, C5, C6, C8. P5's rationale: "C9 needs C4, C5, C6 and C8". Planning order unchanged (C5 is planned in P3). Implementation order unchanged (§8.2).
- **AGENTS:** L1 resolved (D120); D73 "refined by D120, D121, D124"; D39 "refined by D118"; D86 "revised by D89, D92, D126"; D120 "refined by D125, D129"; the open-points rows for C6 (removed again once C6-amended), C9, C10, C11, C12, C13 and C15 updated as in "Recorded for later chunks"; C2's "Recorded for later chunks" C11 line ("group README lookup") moves to C9.

## Open questions

- ✅ **Q1 Highlighter (K22, D118):** starry-night: GitHub's grammars, classes only (the user: GitHub compatibility first).
- ✅ **Q2 Raw HTML (H23, D119):** passed through, sanitised with GitHub's rules, author `class`/`style` dropped.
- ✅ **Q3 Local images and media (L1, D120):** copied into the site as doc assets (§4.6); the CLI writes them in `build`. Refinement approved by the user (D120).
- ✅ **Q4 Links to ignored or missing paths (D121):** unwrapped to text, consistent with "ignored = nonexistent" (D103); reported as `docs/broken-link`.
- ✅ **Q5 GitHub alerts (D122):** supported, with C3-A1.
- ✅ **Q6 Broken links and anchors (D123):** `check` issues, warnings by default (the user's choice, §8.1). Consequence: the hierarchy helper moves to C9 (§8.2); approved by the user.

**From the review (`05a-c5-plan-review.md`), round 3:**

- ✅ **R10 Routes during `check` (D124):** a non-issue: whether a link is broken never depends on routes, so `checkDocs` validates fragments against the target doc's ids whether or not it has a page, and never calls `resolveRoute` (Claude's reframing after the user asked what the problem was; the user agreed).
- ✅ **R25 Doc assets in `dev` (D125):** the CLI keeps a doc-assets directory in its cache and passes `DevContext.docAssetsDir`; the module serves it statically.
- ✅ **R20 Author classes (D126):** stripped; the stale C2/C3/D86 text is fixed.
- ✅ **R1/R2 Footnote ids (D127):** bare `fn-<label>`, with the un-prefix pass (§3.3).
- ✅ **R6 Author anchors (D128):** valid targets, passed as their real element id.
- ✅ **R9 Asset safety (D129):** after the user weighed absolute VCS URLs against copying: copying stays (the site must be portable and the repo may be private; the showcase is built from the local state, since nothing can tell what is or will be committed). Safety: the file and every parent directory resolve, through any symlinks, to a real path inside the repo root, and that real path isn't ignored.
- ✅ **R8 Exact case (D130):** enforced on every filesystem.
- ✅ **R7 Token coverage (D131):** documented; C3's 16 tokens stay.
