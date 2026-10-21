# Chunk plan C5: Markdown pipeline (CLI core)

> Status: **APPROVED** (round 3, 2026-10-19). Q1–Q6 answered (D118–D123); the fresh-eyes review `05a-c5-plan-review.md` triaged and applied (D124–D131). Approved together with C2 amendment B6, C3 amendment C3-A1, the C6 amendment, the C4 note and super-plan R7, all applied. B6's §7 item needed no C2 edit (the stale own-classes text was only in C3 §2.1, fixed by C3-A1). Decisions referenced as `D<n>` are in AGENTS.md §3. **Wording from plan 5 (D201, D202, `04b-c4-redesign-review.md`):** the issue's `file` is `path` (D202; §6's stale "`dir` is the doc's directory" was removed in plan 6); §4.1's exact-case example differs in the base only, and §4.1 and the asset safety check say that a written path resolves in exact case, extension included (D201 governs classification). **Redesign amendment (the representation redesign, `08b-representation-redesign.md`, D160–D205; plan 6 of its application) approved 2026-10-21** (choices O1–O8, D204, D205, the review's Q1–Q3 and Fix 1–28 in `05b-c5-redesign-review.md`): lazy per-doc rendering behind `DocsAccess` (D187); link resolution over the representation (D183) with `RouteTarget`; every element id prefixed with the doc id (D184); `<a>` without `href` for broken and info-less links (C3-A3); `checkDocs` over every `DocRef`; the hierarchy helper and `LinkTargets` removed; see "Redesign amendment" below.

## Goal

Render each document of the representation (a C2 `DocRef`) **on demand** into a `RenderedDoc`: themed `html`, `headings` and `assets` (D23, D71, D120, D187), behind the `DocsAccess` the module receives (`render`, `read`). Markdown gets GFM, GitHub-compatible ids prefixed with the doc id (D184), sanitised raw HTML, class-based syntax highlighting (D39, D118), links resolved to nodes of the representation and turned into URLs by the module (D183) and local media copied into the site (D120). Text docs become an escaped `<pre>`. The same parsing and link resolution also give `check` its `docs/*` issues, over **every** `DocRef` (D123).

C5 renders and checks documents and nothing else. It never decides **which** documents exist (C4 finds the docs and the homes' documents, C9 builds the representation and its `DocRef`s, D179, D185), never calls the module except through a `resolveRoute` callback it is given, and never writes files (C11 writes doc assets).

## 1. Where the code lives

Everything is internal to the CLI package, like C4's `scan/`:

```
packages/tsshowcase/src/markdown/
  link-resolution/ a written link → node target, VCS host, external or broken, over the representation (§4, D183)
  links/          the rehype pass that rewrites href/src (§4, §5)
  ids/            GitHub-compatible heading ids, footnote ids, author anchors, all prefixed with the doc id; RenderedDoc.headings (§3)
  highlight/      code-block highlighting onto tss-token-* classes (§6)
  sanitise/       the raw-HTML schema (§2.2)
  alerts/         GitHub alerts → tss-alert (§2.3)
  text-doc/       DocRef.format 'text' → <pre class="tss-text-doc"> (§7)
  assets/         local media → doc assets: safety checks, hashing, tss-assets/ paths (§4.6)
  docs/           createDocs() (DocsAccess and the asset bookkeeping) and checkDocs(): tie it together (§8)
```

New runtime dependencies of the CLI package (versions fixed at implementation): `unified`, `remark-parse`, `remark-gfm`, `remark-gemoji`, `remark-rehype`, `rehype-raw`, `rehype-sanitize`, `rehype-stringify`, `github-slugger`, `hast-util-to-string`, `unist-util-visit`, and `@wooorm/starry-night` (D118).

## 2. The markdown pipeline

```
remark-parse → remark-gfm → remark-gemoji → remark-rehype (allowDangerousHtml, clobberPrefix: '')
  → rehype-raw → rehype-sanitize (§2.2) → prefix pass (§3, §3.3) → alerts (§2.3) → ids (§3) → links (§4)
  → highlight (§6) → generated classes (§2.1) → rehype-stringify
```

Sanitising comes right after raw HTML is parsed, so every later pass works on trusted structure and its own additions (ids, classes, rewritten URLs) are never stripped.

**GFM features:** tables, task lists (disabled checkboxes), strikethrough, autolinks, footnotes, GitHub emoji shortcodes (`:tada:` → Unicode, no images). **Not supported in v1** (rendered as plain text or as an ordinary code block, documented): math (`$…$`), Mermaid diagrams, GitHub's `@mention`/`#123` autolinks, front matter.

Output is **deterministic** (same inputs, same HTML and `assets`), for reproducible builds.

### 2.1 Classes and attributes in the output (C3 §4)

The module puts doc HTML inside `.tss-content` (C2 rule 6), where the theme styles bare elements (D92), and a content element shares one rule with its class (`.tss-paragraph, .tss-content p`). So C5 adds a class **only where a bare-element selector can't tell elements apart**:

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
- **Author `id`/`name`:** keep the default `user-content-` prefix, so they can't clash with C5's ids, and then get the doc-id prefix like every id (§3); they are valid anchor targets (§3.4, D128).
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

The id set of a doc is everything a `#fragment` may point at: heading ids, footnote ids and author anchors. **Every element id is prefixed with the doc's id** (`DocRef.id`, e.g. `doc-3f2a9c1e`, computed by C9 per representation, D184) and `--`, so ids are unique across all docs and any number of docs can share a page (C2 rule 6; replaces H36's "unique per doc"). Authors still write GitHub-style fragments (`#props`); C5 maps them to the prefixed ids (§4.2).

| Element | Element id (doc `doc-3f2a9c1e`) | A fragment reaching it |
| ------- | ------------------------------- | ---------------------- |
| heading `## Props` | `doc-3f2a9c1e--props` | `#props` |
| footnote `[^note]`, its reference | `doc-3f2a9c1e--fn-note`, `doc-3f2a9c1e--fnref-note` | `#fn-note` (rare) |
| the footnotes label | `doc-3f2a9c1e--footnote-label` | – |
| author anchor `<a name="x">`, `<div id="x">` | `doc-3f2a9c1e--user-content-x` | `#x` or `#user-content-x` (GitHub's) |

**The prefix pass** (right after sanitising, replacing D127's un-prefix pass): every `id` and `name` in the doc, and every token of every `aria-describedby`/`aria-labelledby` value (token lists), gets `<doc-id>--` in front; it touches nothing else, in particular no `href` (Fix 1). **Footnote elements are recognised structurally**, by remark-rehype's `data-footnotes`/`data-footnote-ref`/`data-footnote-backref` markers (read here, removed by §2.1's last pass), never by their id string; only their ids lose the sanitiser's `user-content-` first (they are C5's own, D127), so an author's raw `id="fn-x"` keeps it (Fix 2). Author ids keep `user-content-` (O1); an author `<a name="x">` also gets `id="<doc-id>--user-content-x"`, so it can be found by id (Fix 23). The later ids pass (§3.1) emits already-prefixed slug ids, and the author id it moves into an `<a id>` is already prefixed: nothing is prefixed twice (Fix 3). In-doc `href="#…"` links, footnote references included, are resolved by the links pass (§4.2).

### 3.1 Heading ids

- **The slug follows GitHub** (`github-slugger`, a fresh slugger per doc): lowercase, punctuation dropped, spaces → `-`, duplicates suffixed `-1`, `-2`, …. So `#usage` links written for GitHub keep working, inside the doc and across docs (C5 maps them to `<doc-id>--usage`).
- Every `h1`–`h6`, including raw-HTML headings, gets an id and a permalink `<a class="tss-heading-anchor" href="<route>" aria-label="Link to section: <text>">#</a>` appended. The route is `resolveRoute({ node, doc, anchor })` with the doc's **link-selection node** (the node a link to the doc file selects, §4.1); when that returns `undefined`, the heading gets no permalink (O2).
- **Raw-HTML headings with an author `id`** (R5): the slug id replaces it on the heading itself; the author id moves to an empty `<a id>` at the start of the heading, so links to it keep working (§3.4).
- **Exception:** the footnotes section's hidden label (remark-rehype's `h2#footnote-label`) gets no slug, no permalink and no `RenderedDoc.headings` entry; its id is `<doc-id>--footnote-label`.

### 3.2 `RenderedDoc.headings`

`{ depth, text, id }` in document order, `text` the heading's plain text (`hast-util-to-string`, without the permalink), `id` the prefixed element id. Empty for text docs.

### 3.3 Footnote ids (D127, D184)

remark-rehype builds footnote ids from the **label**, not the visible number: `[^note]` → `fn-note`, references `fnref-note`, repeated references `fnref-note-2`, … (label lowercased and URI-normalised), with `clobberPrefix: ''`. The sanitiser prefixes them with `user-content-`; the prefix pass replaces that with the doc-id prefix (§3) on the ids and the `aria-describedby` references. Footnote links (`href="#fn-note"`, `#fnref-note`) stay as written until the links pass resolves them like in-page anchors (§4.2), through the id set (Fix 1). Tested with labels like `[^Note 1]`.

### 3.4 Author anchors (D128, D184)

Author `id`/`name` attributes (`<a name="x">`, `<div id="x">`) are valid targets, as on GitHub, where `#x` reaches `user-content-x`. In the HTML they become `<doc-id>--user-content-x`; a link `#x` (or `#user-content-x`) that matches one passes that element id as `RouteTarget.anchor`. So an anchor is always an element id present in the doc's HTML, and C10's router just looks it up. When `#x` matches both a heading slug and an author anchor, the heading wins (GitHub's behaviour).

## 4. Links (D183, C2 §5.6)

The CLI decides **which node** a link targets; the module's `resolveRoute` turns the target into a URL (§5). Whether a link is broken never depends on routes (D124).

### 4.1 Resolving a link

Applies to `a[href]`. Media URLs are classified the same way up to step 3, then become doc assets (§4.6).

1. **External:** a scheme (`https:`, `mailto:`, …) or protocol-relative `//` → untouched (`rel="noopener noreferrer"` added; nothing opens in a new tab).
2. **In-page anchor** (`#x`) → the doc itself as the target (§4.2).
3. **Repo path:** resolved against the doc's directory; a leading `/` is the repo root (as on GitHub). Percent-decoded, query and fragment split off. A path normalising outside the repo root → **broken**. (Known false positive, documented: a repo scanned from a monorepo subdirectory (`vcs.directory`) whose READMEs link `../../other/README.md`, which GitHub would resolve.)
4. **The target** (`link-resolution/`, D183's rules in order):
   0. a **directory with a README** stands for that README (C4's `preferredReadme`: markdown first, then name order); a file link is explicit;
   1. **nonexistent or ignored** → **broken** (§4.5);
   2. **a home doc, or a repo or package root directory** → its **home** (`RouteTarget { node: <home id>, doc? }`; a package root in a single-package repo is the repo home, `home:.`);
   3. **out of scope or outside every package** (C4's `RepoScan.inScope`, `packageOf`) → the **VCS host** (§4.4);
   4. **candidates** (read from C9's `PathNodeIndex`, `05b` Q2): for a file, first **its own nodes** (its direct or collection node, configured nodes listing it, except upward listings; they count as primary), else the nodes it is **associated** with (its `DocRef`s, test or showcase associations); for a directory without a README, its **own node** only (an indirect node there, or the root merged from a directory config at its `configuredBy` directory, e.g. `src/` with `scope: ['src']` and a `.tsshowcase` there; the package directory itself is rule 2's; Fix 5). **Selection:** strongest association first (primary > strong > weak), then shallowest depth, then tree order. **Depth** is the nesting depth in the node tree (a package root and homes 0, their children 1, …); **tree order** is pre-order over the D178 default order, homes first, packages in `Representation.packages` order; a duplicate node (D179) falls to tree order (Fix 4). The target is `{ node, doc? }`, `doc` set iff the target file is a doc in the selected node's `docs`. **No candidates:** a file → the VCS host (D73); a directory → **broken**.
5. **The URL:** a node target → `resolveRoute(target)`; `undefined` → the VCS host (§4.4), with the link's fragment kept. A module may answer for a node it doesn't present with another node's URL.

**The link-selection node of a doc** is the node rule 4 selects for the doc file itself (for a home doc, its home, also when a configured node lists it, rule 2 coming first; Fix 6). Its permalinks (§3.1) and its in-page anchors (§4.2) use it.

**Exact case (D130):** matching is case-sensitive, as on GitHub, on every filesystem. Existence uses `RepoFiles.exists` (D108: ignored = nonexistent), and on case-insensitive filesystems (macOS, Windows) the exact spelling of each segment is confirmed with a `list` of its parent; both are tracked reads in `dev`. So `./Readme.md` is missing everywhere when the file is `README.md` (a written path resolves in exact case, extension included; D201 governs classifying the resolved file, not resolving a written path), and the same repo gives the same links, assets and issues on every machine.

### 4.2 Anchors (D184)

- **On a doc target** (an in-page `#x`, or `other.md#x` whose target has `doc` set): the fragment, percent-decoded and then matched exactly, is looked up in the **target doc's id set** (§3) (an empty `#` means the target without an anchor, no issue; Fix 7): a heading slug first, then an author anchor (`x` or `user-content-x`), then a footnote id. Found → passed as `RouteTarget.anchor` (the prefixed element id). Not found → `docs/broken-anchor` (§8.1), and the link goes to the target without an anchor.
- **In-page anchors on a doc shown on several pages (`05b` Q3):** a link whose target is the same doc (an in-page `#x`, a footnote reference or back-reference, a link to the doc's own file with a fragment) also gets `data-anchor="<prefixed id>"` (`tssData.anchor`, C3). Its `href` stays the link-selection node's URL, as the fallback; the module's router scrolls to the element when it is on the current page instead of navigating (C2 rule 6, C10). The doc HTML stays the same on every page (O6).
- **Validation never depends on routes (D124):** the fragment is checked against the target doc's id set whether or not anything has a route; only the URL depends on routes.
- **On other in-app targets** (a node without `doc`, e.g. a link to `Button.tsx#L10`): the fragment is passed to the module as written in `RouteTarget.fragment` (D204, O7), never validated or reported. The module decides: its page URL (using or ignoring the fragment), or `undefined` to send the link to the VCS host with the fragment kept (e.g. line anchors `#L10`, `#L10-L20`; `-base-react`'s default, C10). With no VCS info, `undefined` gives an `unlinked` link (§4.4), so a module returns it only when `representation.home.vcs?.fileUrlTemplate` is set (Fix 15).
- **On VCS host links:** the fragment is passed through unchanged and not validated (`…/blob/<ref>/docs/guide.md#setup`, D205).
- The target doc's id set is computed on demand by parsing it (cached per doc and snapshot), so rendering one doc never requires rendering the others (§8).

### 4.3 Docs whose node has no route

When `resolveRoute` returns `undefined` for a doc's link-selection node, links into it go to the VCS host (rule 5) and its headings get no permalink (§3.1). Its fragments are still validated (D124).

### 4.4 Repo files → VCS host (D24, D110, D205)

`VcsInfo.fileUrlTemplate` with `{path}` filled: the repo-relative path, URL-encoded **per segment** (`encodeURIComponent`, `/` kept); query and fragment from the link appended. Directories use the same template (C4 §9: GitHub and GitLab redirect `blob` to `tree`, Bitbucket's `src` serves both).

**The ref (D205, `05b` Q1):** C4 has substituted `{ref}` in any user template, so C5 fills only `{path}` (Fix 18); filling a VCS URL resolves `vcs.ref` lazily through C4; with the default `CURRENT` and an unreadable `.git` that throws the D205 error. `checkDocs` hits it at the first static VCS link, so `check` (and the check `build`/`dev` run first) fails early.

**No VCS info** (no `VcsInfo.fileUrlTemplate`: no VCS info at all, host `other` without a template, or a ref that couldn't be resolved for the data; Fix 15): the link becomes an `<a>` **without `href`** with `data-link="unlinked"` (C3-A3), its children kept. Not an issue: nothing is broken, the setting is just missing; `checkDocs` counts such links, and C11 prints its one-time `build` warning from that count (D110, O4).

### 4.5 Broken links (D183, revising D121)

A link to an ignored or nonexistent path (D103: ignored = nonexistent), a path outside the repo, or a directory with no candidate node becomes an `<a>` **without `href`** with `data-link="broken"` (C3-A3), its children kept, plus a `docs/broken-link` issue (§8.1). A VCS URL would 404 for a gitignored file, and a relative URL would 404 in the deployed site.

### 4.6 Images and other media: doc assets (L1, D120, D125, D129)

**Why copy:** the showcase must be portable and the source repo may be private, so the site can't point at the VCS host for images (a `blob` URL isn't even an image; a `raw` URL fails for private repos, shows the default branch instead of the working copy, and needs the host to be reachable). And there is no way to know whether a file is or will be committed, or on which branch, so **the showcase is built from the local state**, as it already is for READMEs, tests and showcases (the user's reasoning, D129; the "real path not ignored" condition was Claude's addition, approved with the plan).

**Which URLs:** `img[src]`, `img[srcset]`, `picture > source[srcset]`, `video[src|poster]`, `video > source[src]`, `audio[src]`, `audio > source[src]`, in markdown (`![alt](url)`) and raw HTML alike. Every `srcset` candidate is handled separately, keeping its `1x`/`640w` descriptor. A link (`<a href="./docs/guide.pdf">`) is not media: it still goes to the VCS host (§4.4).

**The contract (C2 §4, amendments B6, B9):**

```ts
export interface RenderedDoc {
  // … html, headings …
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
5. The attribute becomes that path, percent-encoded per segment: `src="tss-assets/3f9a1c0b7d2e4f68/states.png"`. It is **relative to the site root**: under rule 1, `index.html` at the site root is the document every page runs in (hash routing). A module that inserts doc HTML into a document elsewhere sets `<base>` or rewrites the URLs (B6, rule 6). The rendered doc lists `{ source, path }` in `RenderedDoc.assets`, once per path.

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

with `RenderedDoc.assets`:

```ts
[
  { source: 'packages/ui/src/Button/docs/states.png', path: 'tss-assets/3f9a1c0b7d2e4f68/states.png' },
  { source: 'packages/ui/src/Button/docs/dark.png', path: 'tss-assets/91d0e6a2b4c87f13/dark.png' },
  { source: 'packages/ui/src/Button/docs/dark@2x.png', path: 'tss-assets/0c55aa19e3b27d40/dark@2x.png' },
  { source: 'packages/ui/src/Button/docs/light.png', path: 'tss-assets/7be24f0d1a9c3e65/light.png' },
]
```

**Who writes and serves the files (D120, D125):**

- **`build`: the CLI, for rendered docs only (D187).** A doc's assets are written iff `docs.render` was called for it in this build: C5's `createDocs` records every rendered doc's assets (`renderedAssets()`, §8), and C11 writes them (deduplicated by `path`) next to `BuildResult.files` (D64). Modules do nothing. A module file under `tss-assets/` is an error (reserved prefix, rule 1).
- **`dev`: the CLI copies, the module serves a directory.** C11 keeps a doc-assets directory in the CLI cache (`node_modules/.cache/tsshowcase/doc-assets/`, laid out like the site: `tss-assets/<hash>/<name>`) holding the assets of **every** `DocRef` (02b Q2), rebuilt before each `onDataUpdate` from C5's `assetsOf(ref)` (§8, O5) so paths no longer referenced disappear, and passes it as `DevContext.docAssetsDir` (absolute). The module serves that directory statically at `tss-assets/` and nothing else there (`-base-vite`, C15). All path-safety logic stays in the CLI.

**Limits:** SVGs are copied as files and shown through `<img>`, which never runs their scripts. No size limit in v1. A copied file can still be uncommitted, later removed or replaced, exactly like READMEs, tests and showcases (D129).

**Not supported in v1 (documented):** GitHub's `#gh-dark-mode-only` / `#gh-light-mode-only` image fragments (both images show; `<picture>` with `prefers-color-scheme` works, though it follows the OS, not the selected theme).

**External image URLs** (badges) are left alone: they are the README author's own content, like requests made by showcase code; C2 rule 2 gains that exception explicitly (B6).

## 5. The route callback

C5 never sees the module. `createDocs` receives `resolveRoute: (target: RouteTarget) => string | undefined`, bound by C11 to the module, the snapshot's `Representation` and the analysis `state` (C2 §5.6: synchronous, pure). `undefined` means "no in-app page" (§4.1 rule 5), never an error, for any node. A module that throws propagates as a module error naming the module; C11 formats it. **`checkDocs` never calls it** (D124).

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

`DocRef.format: 'text'` (D71; C4's `classifyDoc`: `README`/`README.txt` anywhere, `LICEN[CS]E`, `LICEN[CS]E.txt`, `CHANGELOG` in home roots): `html` is `<pre class="tss-text-doc">` with the source HTML-escaped, no linkification; `headings: []`; `assets: []`. A **markdown licence** (`LICENSE.md`) is a markdown doc like any other: rendered and checked.

## 8. API

```ts
export interface DocsOptions {
  readonly representation: Representation; // the snapshot; DocRefs, nodes, homes (C2 §4)
  readonly scan: RepoScan; // C4: files (existence, case, real paths, binary reads; tracked in dev), inScope, packageOf, vcs
  readonly nodes: PathNodeIndex; // C9, CLI-internal (05b Q2): per file its own nodes (upward listings excluded) and its associated nodes with their levels, per directory its own node; built with the representation
}

export interface CreateDocsOptions extends DocsOptions {
  readonly resolveRoute: (target: RouteTarget) => string | undefined; // §5
}

export interface Docs {
  readonly access: DocsAccess; // C2 §5.8: render(ref) → RenderedDoc (cached per path), read(ref) → the source; rejects for a ref not in this representation
  assetsOf(ref: DocRef): Promise<readonly DocAsset[]>; // dev (O5): the doc's assets without rendering HTML or calling resolveRoute; cached
  renderedAssets(): readonly DocAsset[]; // build (D187): the assets of every doc whose render has resolved through `access`, deduplicated by path; C11 calls it after the module's build settles (Fix 9)
}

/** D187: lazy rendering bound to one representation (a dev snapshot gets its own). */
export function createDocs(options: CreateDocsOptions): Docs;

export interface CheckedDocs {
  readonly issues: readonly Issue[]; // docs/broken-link, docs/broken-anchor; `path` the doc, `line` the link (§8.1)
  readonly unlinkedVcsLinks: number; // links statically bound to the VCS host (rule 3, no candidates) while no VCS info is known (C11's one-time warning, D110, O4); links sent there by resolveRoute returning undefined aren't counted (Fix 11)
}

/** D123, D124: the same parsing, ids and resolution, without routes, hashing or HTML; for `check`, over every DocRef. */
export function checkDocs(options: DocsOptions): Promise<CheckedDocs>;
```

- **Lazy, per doc (D187, O3):** `render(ref)` parses, resolves and renders that doc only; a target doc's id set (§4.2) is computed by parsing the target on demand, cached per doc for the snapshot. Rendering is deterministic, so a cached result equals a fresh one.
- **One resolver (D124):** `createDocs` and `checkDocs` share parsing, ids, the §4.1 resolution and the §4.6 safety checks; whether something is broken never depends on routes. So `check` and the built site always agree on what is broken; only the rendering of correct links depends on routes.
- **Caching and failures (Fix 9, Fix 10):** results are cached per path; a ref is "in this representation" when a `DocRef` with that `path` and `id` exists; a render that rejects (e.g. a throwing `resolveRoute`) isn't cached, so a later call retries, and its doc doesn't count in `renderedAssets()`.
- **Tracked reads (Fix 12):** the sources and asset files C5 reads go through C4's tracked file layer; C11's watcher uses them to know which file changes produce a new snapshot.
- **Each doc once (O6):** a doc path may appear as a `DocRef` in several nodes (with different associations); `checkDocs` checks each path once and `render` caches per path (the HTML doesn't depend on which node refers to it).
- **`dev`:** a change that affects any doc (its source, an asset source, the structure, the analysis) produces a new snapshot with its own `Docs` (C11, 02b H2); nothing is invalidated per doc in v1.

### 8.1 Doc issues in `check` (D123)

`check` reports broken links and anchors as **warnings** (CI sees them; they fail it only when the user raises the severity through `rules` or `--strict=warnings`, D164, D176):

| Rule | Default | When |
| ---- | ------- | ---- |
| `docs/broken-link` | warning | a link or media URL points at an ignored or nonexistent path, a wrongly cased one, a path outside the repo, a directory with no candidate node, or media failing the safety checks (§4.1, §4.5, §4.6) |
| `docs/broken-anchor` | warning | a fragment names no id in the target doc (§4.2) |

- `path` (D202) is the doc, `line` the link's line (best-effort inside raw HTML, whose positions come from parse5), `message` names the URL as written, `hint` says what was looked for (e.g. "no heading `#instalation` in `README.md`; did you mean `#installation`?" when a near match exists). C9 places each issue on every node holding the linking doc's `DocRef` (D180), a home too.
- Which docs: **every `DocRef`** of the representation (home docs included, D123, D183), each path once (O6).
- `docs/*` are CLI rule ids (D175), so `rules` can override them.
- **`build` and `dev` don't report them a second time:** they run the check first (D164, D168). In `dev`, C11 re-runs `checkDocs` for every new snapshot.
- Missing VCS info is not an issue (nothing is broken, the setting is just missing); `build` warns once (C11, D110) from `unlinkedVcsLinks`.

### 8.2 The hierarchy helper (removed)

*Redesign:* the hierarchy helper (super-plan R7) is gone; C9's representation builder (D179) provides the nodes, `DocRef`s and associations C5 resolves links against. C9 still reserves the docs step in `check`, which C5's implementation wires to `checkDocs` (D194: C9 is implemented before C5).

## 9. Tests

Unit tests per internal directory, on in-memory inputs (a fake file layer that can simulate case-insensitive filesystems and symlinks); one fixture for the `check` integration test (§8.1):

- pipeline: a markdown corpus (every GFM feature, emoji, raw HTML, nested lists, tables, footnotes, task lists) rendered to HTML snapshots; determinism (two runs equal); **no `style` attribute** and **no class outside `tssClass`** anywhere in the corpus output;
- sanitising: an XSS corpus (`<script>`, `onerror`, `javascript:` and `data:` URLs, `<iframe>`, `<style>`, SVG with script, `autoplay`, author `class`/`style`/`id`); `language-*` survives to the highlighter;
- ids: GitHub parity on a table of tricky headings (punctuation, emoji, duplicates, non-Latin, inline code); raw-HTML headings with and without an author id; every id prefixed with the doc id (the §3 table), no id left unprefixed and no `user-content-` left on footnote ids; two docs with identical headings rendered side by side have no duplicate ids; footnote ids from labels (`[^note]`, `[^Note 1]`, repeated references), `aria-describedby` intact; the footnote label not in `RenderedDoc.headings`; author anchors (`name`, `id`) reached by `#x` and `#user-content-x`; heading wins over an author anchor;
- links: every rule of §4.1 in order (a directory with a README and with several README forms, nonexistent and ignored, home docs and root directories incl. the single-package home, out of scope and outside packages, own nodes vs associations, upward listings ignored, the selection order by association, depth and tree order, `doc` set or unset, no candidates for a file and for a directory, `resolveRoute` returning `undefined`); root-relative and `..` paths, out-of-root paths, percent-encoding, query and fragment, exact case on a simulated case-insensitive filesystem; broken links as `<a data-link="broken">` without `href`; no VCS info as `<a data-link="unlinked">` without `href`;
- VCS: links with the resolved ref (`HEAD`, a SHA, a tag; D205) for GitHub, GitLab and Bitbucket (`src/<ref>/…`), with and without `vcs.directory`, a user template with and without `{ref}`, segment encoding (spaces, `#`, Unicode) (C4 "Recorded for later chunks");
- anchors: in-page, cross-doc, the lookup order (heading, author anchor, footnote), unknown fragment (issue, link without anchor), fragments passed as `RouteTarget.fragment` on non-doc in-app targets (kept on the VCS URL when the module returns `undefined`), kept unvalidated on VCS URLs; a doc whose link-selection node has no route (no permalinks, links to the VCS host, fragments still validated);
- resolution edge cases (Fix 5, 6, 16): a link to a collection member (its collection's node) and to the virtual `<dir>/<key>` path (broken); to a test and a showcase file (associated nodes, `fragment` passed); to an exporting file both listed by a configured node and with its direct node; to a `configuredBy` directory; into a skipped package (broken); a doc that is a `DocRef` in several nodes; a home doc also listed by a configured node; a doc listed by two configured nodes; `resolveRoute` throwing (the render rejects, isn't cached); `rel="noopener noreferrer"` on external links;
- ids (Fix 2, 23, 16): `name` and `aria-labelledby` prefixed (token lists); a raw-HTML `id="fn-x"` keeps `user-content-`; an author `<a name>` found by its prefixed id; percent-encoded fragments (Fix 7);
- `assetsOf(ref)` deep-equals `render(ref).assets` (Fix 8); `renderedAssets()` ignores rejected and unresolved renders (Fix 9);
- in-page anchors: `data-anchor` on same-doc links (in-page, footnote references and back-references, the doc's own file with a fragment), not on links to other docs;
- laziness: rendering one doc parses only it and the docs its anchors point into; `render` cached per path; `assetsOf` without HTML or routes; `renderedAssets()` lists exactly the rendered docs' assets; a ref not in the representation rejects;
- assets: the §4.6 example verbatim; dedup across docs; same content under two names; a changed file gets a new path; missing, ignored, wrongly cased and out-of-repo media; a symlinked file and a symlinked parent directory leading outside the repo; an in-repo symlink to an ignored file; a directory as `src`; SVG; `srcset` with one bad candidate; percent-encoded file names; text docs have none;
- `checkDocs`: every broken case yields exactly the issue `render` acted on (same resolver), regardless of routes; every `DocRef` checked, each path once; `unlinkedVcsLinks` counted; near-match hints; markdown licences and home docs;
- highlighting: the mapping covers every class in starry-night's CSS, `diff` → inserted/deleted, unknown language, TSX sample snapshot;
- alerts: all five kinds, lower-case markers, a marker with trailing text, a nested blockquote, an ordinary blockquote;
- text docs: escaping (`<`, `&`), no headings, no assets.

Fixtures: one for the `check` integration test (§8.1), with broken links and anchors, a directory config and a home doc linking into the tree.

## 10. Documentation

- CLI README, "How READMEs are rendered": supported markdown (GFM, emoji, alerts), what isn't (math, Mermaid, mentions, `#gh-dark-mode-only`), raw HTML and what the sanitiser removes (including author classes), how each kind of link is resolved (the §4.1 rules in user terms: which page a link opens, when it goes to the VCS host, when it is broken), exact case, element ids prefixed with the doc id while authors keep writing GitHub fragments, local images copied to `tss-assets/` from the local state and the safety rules, external images, highlighted languages and the token mapping with its coverage (D131), heading ids matching GitHub, VCS links pinned to `vcs.ref` (default the checked-out commit, D205) with its caveats, the `vcs.directory` false positive; the `docs/*` rules in the rule list.
- The `catalogue/` README (C3's generator): which classes and attributes C5 emits (§2.1), that bare elements inside `.tss-content` cover the rest, and which syntax tokens starry-night leaves unused.
- `CONTRIBUTE.md` glossary: doc, doc id, link-selection node, doc asset, id set.

## Acceptance criteria

- The corpus renders to the reviewed snapshots; output contains no `style` attribute and no class outside the catalogue.
- Every rule of §4.1, every anchor case and every host's link with each ref form (D205) is covered by a test.
- The XSS corpus renders without any script, handler, `data:` or `javascript:` URL.
- The §4.6 example produces exactly the documented HTML and `RenderedDoc.assets` (with the real hashes of the test files); no symlink case publishes a file outside the repo or an ignored file.
- `checkDocs` and `render` agree on every broken case; rendered HTML has no duplicate id across any two docs.
- `npm run ci` passes.

## Out of scope

Finding docs and homes (C4); the representation, doc ids and the docs step's place in `check` (C9); calling `createDocs` per snapshot, writing doc assets in `build`, the `dev` doc-assets directory, printing the `no-vcs` warning (C11); embedding rendered HTML in the bundle and serving `docAssetsDir` in `dev` (C15); scrolling to ids in the app (C10); the styles for alerts, doc elements and link states (C3-A1, C3-A3).

## Recorded for later chunks

- **C9:** the CLI-internal `PathNodeIndex` built with each representation (paths → own nodes with upward listings excluded, associated nodes with levels, a directory's own node), handed to C5 and usable for issue placement (D180) (`05b` Q2); ~~the shared hierarchy helper~~ (replaced by the representation builder, D179); doc ids per representation (D184); a reserved docs step in `check`, wired by C5's implementation to `checkDocs` over every `DocRef`; `docs/*` issues placed on every node holding the linking doc (D180).
- **C10:** its router keeps links marked `data-anchor` on the current page when the element is there (scroll, `aria-current="location"`), else follows the `href` (`05b` Q3); `-base-react`'s `resolveRoute` sends node targets with a line-anchor `fragment` (e.g. `L10`, `L10-L20`, `lines-10`; C10 finalises the forms, Fix 19) to the VCS host (`undefined`) when `representation.home.vcs?.fileUrlTemplate` is set (Fix 15), else to the node's page (D204); the router scrolls to any element id in a doc's container (all prefixed with the doc id, D184); each doc rendered at most once per page (C2 rule 6, C3-A3); `data-link` states on its own links that can't link.
- **C11:** `createDocs` per data assembly and per `dev` snapshot, bound to the module (`resolveRoute`), the representation and `state`; the one-time `no-vcs` warning from `checkDocs`'s `unlinkedVcsLinks` (D110); **writes the assets of rendered docs** (`renderedAssets()`, deduplicated by `path`) next to `BuildResult.files` and rejects module files under `tss-assets/`; the "no absolute build-machine paths" output check skips doc assets (user content, binary); keeps the `dev` doc-assets directory in the CLI cache with every `DocRef`'s assets from `assetsOf`, rebuilt before each `onDataUpdate`, passed as `DevContext.docAssetsDir` (D125, 02b Q2); re-runs `checkDocs` per snapshot.
- **C12:** a README image in `build` and in `dev` (updated when the image changes), footnote and cross-doc anchor scrolling, author anchors, themed alerts, broken and unlinked link states.
- **C13:** README authoring notes (what renders like GitHub, what doesn't; local images and their safety rules; exact case; the `docs/*` rules; GitHub fragments mapped to prefixed ids, D184; where links lead, D183).
- **C15:** in `dev`, serves `DevContext.docAssetsDir` statically at `tss-assets/` and nothing else under that prefix; how rendered HTML is embedded in the bundle, so only rendered docs reach the output (D187, E2).
- **v1 cut line (K21, D98):** candidates: alerts, `video`/`audio`, near-match hints (R38).

## C2 amendment B6 (approved with this plan, applied)

> History: superseded where the redesign amendment says so (`Doc` → `RenderedDoc`, `heading` → `RouteTarget.anchor`, unwrapping → `<a>` without `href`, the hierarchy helper replaced by C9's representation builder; D183, D184, D187).

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

> History: superseded where the redesign amendment says so (`Doc` → `RenderedDoc`, `heading` → `RouteTarget.anchor`, unwrapping → `<a>` without `href`, the hierarchy helper replaced by C9's representation builder; D183, D184, D187).

- **C5:** `check` issues `docs/broken-link`, `docs/broken-anchor` (D123, D124); doc assets copied by the CLI (L1, D120, D125, D129); alerts (D122); starry-night (D118); depends on C2, C3, **C4** (the file layer).
- **C2 CLI duties table:** "Building the hierarchy from flat units + `unitRoot` (groups, nesting, group READMEs)" moves from C11 to **C9 (shared helper, reused by C11)**; new rows "writing doc assets into the output" → C11 and "the `dev` doc-assets directory" → C11.
- **C9:** the hierarchy helper and group README lookup (moved from C11's "Group READMEs (D78)" bullet); a reserved docs step in `check`; depends on C4, C5, C6, C8. **C11:** writes doc assets, maintains the `dev` doc-assets directory. **C15:** serves `docAssetsDir`. **C12:** the C5 E2E items.
- **Dependency edges:** C5 → C2, C3, C4; C9 → C4, C5, C6, C8. P5's rationale: "C9 needs C4, C5, C6 and C8". Planning order unchanged (C5 is planned in P3). Implementation order unchanged (§8.2).
- **AGENTS:** L1 resolved (D120); D73 "refined by D120, D121, D124"; D39 "refined by D118"; D86 "revised by D89, D92, D126"; D120 "refined by D125, D129"; the open-points rows for C6 (removed again once C6-amended), C9, C10, C11, C12, C13 and C15 updated as in "Recorded for later chunks"; C2's "Recorded for later chunks" C11 line ("group README lookup") moves to C9.

## Redesign amendment (plan 6 of the representation redesign's application; approved 2026-10-21)

Plan 6 of the redesign's application (`08b-representation-redesign.md` §7.7, §7.10, §12; `08e` E7), after AGENTS, C2 B9/B10, C1 A4, C3-A3 and C4's amendment; the choices O1–O8 and the review are in `05b-c5-redesign-review.md`. C5 now renders `DocRef`s of the representation lazily instead of a whole `Doc` set, and resolves links to nodes:

- **Goal, §1:** `RenderedDoc` per `DocRef` behind `DocsAccess` (D187); docs come from C4 and C9; `link-targets/` → `link-resolution/`, `render-docs/` → `docs/`.
- **§2.1, §2.2:** "doc HTML"; author ids get the doc-id prefix.
- **§3:** every element id prefixed `<doc-id>--` (D184), the prefix pass replacing D127's un-prefix pass, footnote and author-anchor ids (O1); permalinks to the link-selection node, none without a route (O2); `RenderedDoc.headings`.
- **§4:** D183's resolution (rules 0–5) replacing the unit/package/group table; the link-selection node; anchors on doc targets as `RouteTarget.anchor`, lookup order; on other in-app targets the fragment goes to the module as `RouteTarget.fragment` (D204, O7); `<a>` without `href` with `data-link="broken"` / `"unlinked"` (C3-A3) instead of unwrapping (D121); §4.3 docs whose node has no route.
- **§4.6:** `RenderedDoc.assets`; assets written for rendered docs only (D187); `dev` pre-copies every `DocRef`'s assets through `assetsOf` (02b Q2, O5).
- **§5:** the callback takes a `RouteTarget`; `undefined` is never an error.
- **§7:** text docs per `DocRef.format` and C4's `classifyDoc`.
- **§8:** `createDocs` (`access`, `assetsOf`, `renderedAssets`) and `checkDocs` returning `{ issues, unlinkedVcsLinks }` (O3, O4); lazy per doc; each path once (O6); `LinkTargets`, `DocInput`, `isScanned`, `RenderedDocs` removed; §8.1 every `DocRef`, placement by C9, `path` (D202), the stale `dir` sentence removed; §8.2 the hierarchy helper removed.
- **§9, §10, acceptance, out of scope, recorded points:** updated.
- **Review outcomes (`05b` "Agent review"):** Q1 the `vcs.ref` error fires lazily, first in `checkDocs`; Q2 C9's CLI-internal `PathNodeIndex`; Q3 `data-anchor` on in-page anchors; Fix 1–28 (footnote passes, depth and tree order, fragment matching, caching and failures, tests, stale `HEAD` wording, seams).
- **Also decided in plan 6:** D204 (`RouteTarget.fragment`, O7) and D205 (`vcs.ref`, default `CURRENT`; applied to C4 and C2, C5's tests only).
- **Unchanged:** the pipeline (§2), raw HTML (§2.2) apart from ids, alerts (§2.3), media safety and hashing (§4.6), VCS URLs (§4.4) apart from the info-less case, highlighting (§6), text-doc rendering (§7).

## Open questions

- ✅ **Q1 Highlighter (K22, D118):** starry-night: GitHub's grammars, classes only (the user: GitHub compatibility first).
- ✅ **Q2 Raw HTML (H23, D119):** passed through, sanitised with GitHub's rules, author `class`/`style` dropped.
- ✅ **Q3 Local images and media (L1, D120):** copied into the site as doc assets (§4.6); the CLI writes them in `build`. Refinement approved by the user (D120).
- ✅ **Q4 Links to ignored or missing paths (D121)** *(history: since D183 an `<a>` without `href`, `data-link="broken"`, §4.5)*: unwrapped to text, consistent with "ignored = nonexistent" (D103); reported as `docs/broken-link`.
- ✅ **Q5 GitHub alerts (D122):** supported, with C3-A1.
- ✅ **Q6 Broken links and anchors (D123):** `check` issues, warnings by default (the user's choice, §8.1). Consequence: the hierarchy helper moves to C9 (§8.2); approved by the user. *(History: the helper is replaced by C9's representation builder, D179; every `DocRef` is checked, D183.)*

**From the review (`05a-c5-plan-review.md`), round 3:**

- ✅ **R10 Routes during `check` (D124):** a non-issue: whether a link is broken never depends on routes, so `checkDocs` validates fragments against the target doc's ids whether or not it has a page, and never calls `resolveRoute` (Claude's reframing after the user asked what the problem was; the user agreed).
- ✅ **R25 Doc assets in `dev` (D125):** the CLI keeps a doc-assets directory in its cache and passes `DevContext.docAssetsDir`; the module serves it statically.
- ✅ **R20 Author classes (D126):** stripped; the stale C2/C3/D86 text is fixed.
- ✅ **R1/R2 Footnote ids (D127)** *(history: since D184 prefixed with the doc id, §3.3)*: bare `fn-<label>`, with the un-prefix pass.
- ✅ **R6 Author anchors (D128):** valid targets, passed as their real element id (since D184 prefixed, as `RouteTarget.anchor`).
- ✅ **R9 Asset safety (D129):** after the user weighed absolute VCS URLs against copying: copying stays (the site must be portable and the repo may be private; the showcase is built from the local state, since nothing can tell what is or will be committed). Safety: the file and every parent directory resolve, through any symlinks, to a real path inside the repo root, and that real path isn't ignored.
- ✅ **R8 Exact case (D130):** enforced on every filesystem.
- ✅ **R7 Token coverage (D131):** documented; C3's 16 tokens stay.
