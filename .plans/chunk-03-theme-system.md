# Chunk plan C3: Theme system + default themes

> Status: **APPROVED** (round 6, 2026-10-18). Reworked after two fresh-eyes reviews (`03a-c3-plan-review.md`, `03b-c3-plan-review.md`) and the isolation redesign (D89: Shadow-DOM shell, light-DOM showcases). Decisions referenced as `D<n>` are in AGENTS.md §3. Approving this plan also closes the C2 plan (C3 fills its catalogue slot, D28).

## Goal

Design the theme ⇄ module contract that C2 left as a slot:

- the **isolation model** between the themed shell and the user's showcases (D89);
- the **CSS variable hierarchy** (layered, so overriding a base token cascades);
- the **class catalogue** (`tss-…`), with states and variants as ARIA/data attributes (D84);
- the **scoping transform** the CLI applies to theme CSS (D65, D82);
- the **lint package** that enforces the styling policy (D46, D88);

and ship the **default light and dark themes** in `@ofzza/tsshowcase-theme-default`.

## Requirements this plan answers

From REQUIREMENT.md: every styling value a theme might want to change is a variable; derived variables extend from base ones ("override primary and secondary colours, or the entire palette, or the base spacing, or the entire spacing hierarchy"); documented class names that modules use; **no styling unless it uses an exposed variable or class, or is derived directly from an exposed variable**. Plus D7 (several themes, runtime switch), D28 (catalogue as constants in `-api`), D39 (class-based syntax highlighting), D45 (`tss-` prefix), D46 (one enforced styling policy), D65 (unscoped theme CSS, composition).

Deferred points from the super-plan review, now settled elsewhere:

- "How the default is picked with more than two themes" → C2 inclusion rule 4.
- "Are user showcase components bound by the theme rules?" → no (D46); and with D89 the theme can't reach them at all.

## 1. The isolation model (D89)

Earlier rounds tried to share one document between the themed shell and the user's showcases, and kept running into leaks in both directions, specificity contests, inherited typography and opt-in/opt-out ceremony. D89 separates them instead.

**Principle:** we control how the _shell's_ CSS is injected; we don't control how a _user's_ CSS is injected (Vite and CSS-in-JS libraries put it in `document.head`; portals put content in `document.body`). So the shell is encapsulated, and the user's world is the ordinary document.

- **The shell lives in a shadow root.** The module renders the whole app (header, nav, pages, README content, test results, theme switcher) inside a shadow root attached to a host custom element with a reserved name, **`<tsshowcase-app>`** (an `-api` inclusion constant, so ordinary element selectors in user CSS never match it, H2). The theme's shell stylesheet and the module's own CSS are **adopted into that shadow root** (`adoptedStyleSheets = [themeShell, ...moduleSheets]`, in that order, H21), never added to `document.head`. The shell's root element inside the shadow root carries the class `tss-shell` and the theme attribute (`data-theme`, §5).
- **Showcases live in the light DOM.** Each showcase is rendered into a light-DOM **wrapper element** `<div data-tss-showcase>` (a child of the host), which is assigned to a **named slot** (`slot="tss-showcase-<n>"`, H13) inside its `tss-showcase-stage` (in React: a `createPortal` from the shell tree into the wrapper, so context and events behave normally).
- **What that guarantees:**
  - **theme → showcase:** selectors in the shadow root can't match light-DOM content (the theme may not use `::slotted`, `slot`, `:host()`, `:host-context()` or `::part`, §2.1), so nothing leaks into a showcase;
  - **user CSS → shell:** document selectors can't match shadow content, so a user's globals and resets never touch the shell's elements or README content;
  - **portals:** modals, dropdowns and tooltips land in `document.body`, where the user's CSS applies and no theme CSS exists;
  - **no clipping**, given the layout rules below: showcases are real page content, so overlays float over the whole page.
- **Inheritance across the boundary** (inheritance follows the flattened tree, so a slotted element inherits from its slot):
  - the module's fixed shell CSS sets `:host { all: initial !important; display: block !important; }`: document rules beat `:host` rules unless they're `!important` (H2), and this stops a user's `body { font-family: … }` or `*`/`all: unset` resets from reaching the shell;
  - the showcase slot gets `slot { all: initial !important; display: contents !important; color-scheme: <selected scheme> !important; }`, so the shell's typography and colours don't reach the showcase, while native controls follow the showcase's selected light/dark mode;
  - custom properties are not reset by `all`, so `var(--tss-…)` remains _available_ to a showcase's inline content.
- **Typography is the showcase's own (H1).** Because the slot resets inheritance, a showcase doesn't inherit the user's `body` typography either; unstyled text would fall back to browser defaults (typically a serif). So:
  - the `[data-tss-showcase]` wrapper plays the role `body` plays in the user's app, and is documented as such;
  - modules offer a **showcase stylesheet** option (React: `moduleOptions.showcaseStylesheets: ['./src/showcase.css']`), CSS injected into the **document** (Storybook-preview style), where users put their app-level typography, e.g. `body, [data-tss-showcase] { font-family: …; color: …; }`;
  - with that, a showcase looks as in the user's app; without it, inline text uses browser defaults while portals get the document's `body` styles. D89 doesn't claim more than this.
- **What user CSS must not do** (documented in C13, H2/H3): style `html`/`body` layout (`display: flex`, `height: 100%`, `overflow: hidden`, …) or the root font size (`html { font-size: 62.5% }`). Such rules apply to the real page that contains the shell, and `rem`-based theme tokens follow the root font size (H3, a pitfall).
- **Layout rules that keep showcases unclipped (H5):**
  1. the shell uses **document scrolling**: no scroll container on any ancestor of `tss-showcase-stage`;
  2. neither themes nor modules set `overflow` (other than `visible`), `transform`, `filter`, `backdrop-filter`, `contain`, `container-type` or `will-change` on stage ancestors (documented per class; linted where feasible);
  3. the host establishes a stacking context (`isolation: isolate`), so the whole shell paints below body-level portals; `--tss-z-*` are relative to the shell.
- **The stage** is themed (border, padding, and a backdrop matching the showcase's selected scheme via `data-scheme`, §4), so a showcase that sets no background of its own sits on a suitable light or dark backdrop.
- **Colour scheme for showcases:** every showcase receives the selected scheme (`'light' | 'dark'`). Each module defines its showcase component interface (for React, e.g. `export default function Showcase({ colorScheme }: { colorScheme: 'light' | 'dark' })`) and pairs every showcase with its own light/dark selector (§10). The prop is the reliable channel: portalled content (in `body`) gets neither the slot's `color-scheme` nor the `--tss-*` variables (H10).
- **Showcases of shell components** (our own D41 showcases of `-base-react` components) need the theme. They opt into rendering **inside** the shadow root instead of being slotted, as a whole-showcase switch (module-defined, e.g. `export const shell = true`). The opt-in is only for components styled purely by catalogue classes and variables: CSS a showcase imports lands in `document.head` and doesn't reach inside the shadow root (H11). For such a showcase the scheme selector sets the configured theme matching the selected scheme on its stage.
- **Shell overlays** (e.g. a switcher dropdown) render inside the **`.tss-shell` element**, so they get the theme (H29); never into `document.body`.
- **Document-level styles:** only (a) the host element's own box, (b) the **fonts stylesheet** (§5), (c) the user's showcase stylesheets, and (d) one documented exception for the page itself (H6): the module sets `color-scheme` and the canvas background on `<html>` from the active theme (kept in sync on switch), so scrollbars and the overscroll canvas follow a dark theme; the built `index.html` ships `<meta name="color-scheme" content="light dark">` and a tiny inline script applying rule 4's initial selection to `<html>` before the bundle loads, so there's no white flash.
- **Headings and anchors (H9):** fragment navigation and `:target` don't reach shadow trees. The module's router scrolls to and focuses the heading named by the route inside the shadow root (looking it up within the doc's container, since heading ids are unique per doc only, H36) and marks it `aria-current="location"`, which themes style instead of `:target`.
- **Events and accessibility (H14):** events from the shell are retargeted to the host, so the module stops key-event propagation from shell inputs at the host (user hotkey handlers shouldn't fire while typing in the shell); ARIA references can't cross the boundary, so the stage is labelled from inside the shadow tree.
- **Fallback (H15):** where constructable stylesheets are unavailable (e.g. jsdom in tests), the runtime falls back to `<style>` elements in the shadow root, in the same order.

**Binding vs free (H34, C2 rules 3 and 7):** binding for every module: the shell lives in a shadow root, the theme stylesheets are included as specified, and theme CSS never applies to showcase content. Free: how showcases are isolated from that shadow root (slots as above, or e.g. iframes), provided portals work and a colour scheme is passed. `react-default` uses the technique above (C10, C15).

## 2. Division of labour

| Concern                                                                                               | Owner                                                                              |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Look:** colour, typography, spacing, borders, radii, shadows, motion of the shell and README content | the **theme**: it may style anything inside the shell (§2.1)                       |
| **Arrangement:** where things go (grid areas, flex direction, sticky nav, sizes from component tokens) | the **module**: its CSS uses only structural keywords and `var(--tss-…)` (D46)     |
| **Names:** every class, variable and data attribute value                                             | the **catalogue** in `-api` (D28), documented once                                 |
| **Scoping** of theme CSS under `data-theme`, `color-scheme`, assembling the stylesheets                 | the **CLI** (D82, §5)                                                              |
| **Isolation:** host element, shadow root, slots, adopting stylesheets, scheme selectors, document-level page styles | the **module** (§1): `-base-vite` (C15) provides a framework-agnostic runtime API (adopt sheets into a given shadow root, theme state `getInitialTheme`/`setTheme`/`subscribe`, document-level `color-scheme`, dev stylesheet updates); `-base-react` (C10) owns the host, slot portals, overlays, heading scroll, and renders `data-theme` from that state (H28) |

**Property ownership is documented per class** (G12): for every shell and showcase class, the catalogue docs state which properties the **module** sets (arrangement, using component tokens such as `--tss-nav-width`, `--tss-header-height`, `--tss-content-max-width`) and which the **theme** sets (look). Both stylesheets live in the same shadow root; theme class rules carry no extra specificity from scoping (§5), so source order decides: theme first, module CSS after it.

### 2.1 What a theme may do (D86, revised by D89)

A theme is just CSS, and it may style **anything inside the shell**:

- set or override any catalogue variable, at any layer (§3);
- style any catalogue class, and bare elements anywhere in the shell (only the shell lives in the shadow root);
- **introduce its own variables, classes and selectors** when the catalogue isn't enough; a theme that composes this one (C2 §7) gets them too and can override them.

Ideally the catalogue is enough for any theme; extending it is allowed, not expected. The rules, which keep themes interchangeable:

1. **No self-scoping, no document selectors and nothing that reaches showcases:** no `[data-theme]`, `:host`, `:host()`, `:host-context()`, `html`, `body`, `slot`, `::slotted()` or `::part()` selectors, no compound `:root`, and no `color-scheme` declarations (they come from `ThemeDefinition.scheme`, H22). The CLI scopes (§5); the theme only ever applies inside the shell and never to showcase content (H4).
2. **Own names use their own prefix,** never `tss-` / `--tss-`, which are reserved for the catalogue, so a future catalogue addition never collides with a theme's invention (e.g. `--ocean-wave-height`, `.ocean-ribbon`, `@keyframes ocean-pulse`, `font-family: 'Ocean Sans'`). Keyframe names always use the theme's own prefix (the catalogue defines none, H26).
4. **Respect the layout rules of §1** (no clipping or containing-block properties on stage ancestors).
3. **Modules only use catalogue names in their markup,** plus their own prefixed classes for arrangement only, which themes never style (G25). A theme's own classes are therefore useful in two places only: showcases rendered inside the shell, and raw HTML in READMEs (whether C5 passes raw HTML through, sanitised, is a C5 decision, H23).

These are contract rules for every theme (rules 1–4), documented each with its purpose in **"Building a theme that works with any module"** (§11). The `-lint` presets check them (§9); they're enforced on our own themes only (D46).

**"Split into tokens, light/dark and catalogue"** is only how the _default_ theme organises its files (§7), so a composing theme can reuse pieces (named exports, §7).

## 3. The variable hierarchy

Four layers. Each layer is defined **only** in terms of the layer above it (plus, for base tokens, literal values). Overriding any token recomputes everything below it.

```
base tokens ──► derived scales ──► semantic tokens ──► component tokens
(literals)      (calc/color-mix)    (pick from scales)   (pick from semantic/scales)
```

**Units:** the default theme sizes type and spacing in `rem`, which respects the reader's browser font-size setting; the price is that a showcase setting the root font size rescales the shell (a documented pitfall, H3).

**Naming:** `--tss-<category>-<name>[-<step>]`. Constants in `-api` use camelCase keys (`tssVar.colorPrimary300` → `'--tss-color-primary-300'`).

### 3.1 Base tokens (the knobs; literal values live only here)

| Category       | Tokens                                                                                                                                                                                                                       |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Brand colours  | `--tss-color-primary`, `--tss-color-secondary`                                                                                                                                                                               |
| Neutral        | `--tss-color-neutral` (a colour: the grey the neutral ramp is built from), `--tss-color-light`, `--tss-color-dark` (ramp endpoints, e.g. white and near-black)                                                              |
| Status colours | `--tss-color-success`, `--tss-color-warning`, `--tss-color-danger`, `--tss-color-info`                                                                                                                                      |
| Typography     | `--tss-font-family-sans`, `--tss-font-family-mono`, `--tss-font-size` (base, `1rem`), `--tss-type-ratio` (e.g. `1.25`), `--tss-line-height`, `--tss-font-weight-normal`, `--tss-font-weight-medium`, `--tss-font-weight-bold` |
| Spacing        | `--tss-space` (the base unit, e.g. `0.25rem`)                                                                                                                                                                                |
| Shape          | `--tss-radius` (base), `--tss-radius-full` (pill radius; a base token so square-cornered themes keep pills, G15), `--tss-border-width`, `--tss-focus-width`                                                                  |
| Elevation      | `--tss-shadow-color`                                                                                                                                                                                                         |
| Opacity        | `--tss-opacity-muted`, `--tss-opacity-disabled` (G8)                                                                                                                                                                         |
| Motion         | `--tss-duration` (base), `--tss-easing`                                                                                                                                                                                      |
| Layering       | `--tss-z-nav`, `--tss-z-overlay`                                                                                                                                                                                             |

### 3.2 Derived scales (formulas, scheme-agnostic)

| Scale         | Tokens                                                                                                                       | Formula                                                                                                                                                  |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Colour ramps  | `--tss-color-<hue>-50 … -950` for `primary`, `secondary`, `neutral`, `success`, `warning`, `danger`, `info` (11 steps each) | `500` = the base colour; lighter steps `color-mix(in oklch, var(--tss-color-<hue>), var(--tss-color-light) N%)`, darker steps mix with `--tss-color-dark` |
| Type scale    | `--tss-font-size-xs`, `-sm`, `-md`, `-lg`, `-xl`, `-2xl`, `-3xl`, `-4xl`                                                      | `md` = `--tss-font-size`; each step up multiplies by `--tss-type-ratio`, each step down divides (nested `calc()`)                                        |
| Line heights  | `--tss-line-height-tight`, `-normal`, `-loose`                                                                               | from `--tss-line-height`                                                                                                                                 |
| Spacing scale | `--tss-space-1 … -8`                                                                                                          | `calc(var(--tss-space) * n)` for n = 1, 2, 3, 4, 6, 8, 12, 16 (so `--tss-space-5` is 6×; the generated docs list every multiplier)                       |
| Radii         | `--tss-radius-sm`, `-md`, `-lg`                                                                                              | `× 0.5`, `× 1`, `× 2` of `--tss-radius`                                                                                                                  |
| Shadows       | `--tss-shadow-sm`, `-md`, `-lg`                                                                                              | offsets and blur from the spacing scale, colour from `--tss-shadow-color` via `color-mix`                                                               |
| Durations     | `--tss-duration-fast`, `-normal`, `-slow`                                                                                    | `× 0.5`, `× 1`, `× 2` of `--tss-duration`                                                                                                                |

**Reduced motion (G16):** the `@media (prefers-reduced-motion: reduce)` block that zeroes the durations sits in the **last** default file (`catalogue.css`), so later base or derived duration values can't override it by source order. Composed themes that redefine durations must repeat it last (documented with the §3.5 gotcha).

### 3.3 Semantic tokens (scheme-specific: what light and dark disagree on)

- **Surfaces:** `--tss-surface`, `--tss-surface-raised`, `--tss-surface-sunken`, `--tss-surface-inverse`.
- **Text:** `--tss-text`, `--tss-text-muted`, `--tss-text-subtle`, `--tss-text-inverse`, `--tss-text-on-accent`.
- **Lines:** `--tss-border`, `--tss-border-strong`.
- **Interaction:** `--tss-accent`, `--tss-accent-hover`, `--tss-link`, `--tss-link-hover`, `--tss-link-visited`, `--tss-focus-ring`, `--tss-selection`.
- **Code:** `--tss-code-surface`, `--tss-code-text`, and syntax colours `--tss-syntax-<token>` for `comment`, `keyword`, `string`, `number`, `function`, `type`, `variable`, `constant`, `operator`, `punctuation`, `tag`, `attribute`, `property`, `regexp`, `inserted`, `deleted` (D39).
- **Test status:** `--tss-status-<s>` and `--tss-status-<s>-surface` for every `data-status` value: `passed`, `failed`, `skipped`, `todo`, `running`, `queued`, `unavailable`, `not-run` (G14).
- **Showcase stage:** `--tss-stage-surface-light`, `--tss-stage-surface-dark` (the stage backdrop for each selected showcase scheme, §1).

Light picks e.g. `--tss-surface: var(--tss-color-neutral-50)`; dark picks `var(--tss-color-neutral-950)`. Same ramps, different picks.

### 3.4 Component tokens (scheme-agnostic, per catalogue area)

Defined from semantic tokens and scales, so a theme can retune one component without touching the rest. Initial set (completed during implementation, reviewed with the user):

- **Headings:** `--tss-heading-color`, `--tss-heading-font-weight`, `--tss-heading-line-height`, `--tss-heading-1-size … -6-size`, `--tss-heading-spacing`.
- **Body text:** `--tss-paragraph-spacing`, `--tss-content-max-width`.
- **Code:** `--tss-code-font-size`, `--tss-code-block-padding`, `--tss-code-radius`.
- **Tables, quotes, rules:** `--tss-table-cell-padding`, `--tss-table-border`, `--tss-table-stripe`, `--tss-blockquote-border`, `--tss-blockquote-text`, `--tss-rule-color`.
- **Shell:** `--tss-header-height`, `--tss-header-surface`, `--tss-nav-width`, `--tss-nav-surface`, `--tss-nav-link-padding`, `--tss-nav-link-active-surface`, `--tss-page-padding`.
- **Showcase:** `--tss-showcase-border`, `--tss-showcase-radius`, `--tss-showcase-stage-padding`, `--tss-showcase-error-surface`.
- **Tests, badges, switchers:** `--tss-test-row-padding`, `--tss-badge-padding`, `--tss-badge-radius`, `--tss-switcher-*`.

### 3.5 How overriding cascades, and its one gotcha

All four layers are declared on the **theme scope element**, the shell root `.tss-shell[data-theme="…"]` inside the shadow root (§5). Because every derived, semantic and component token is a `var()` expression declared on that same element, overriding a base token in a theme file recomputes everything below it:

- override `--tss-color-primary` → its ramp, the accent, links, active nav, … all follow;
- override `--tss-color-primary-600` → only things picking that step follow;
- override `--tss-space` → the whole spacing scale and every padding/gap built on it follows;
- override `--tss-space-1 … -8` individually → a hand-tuned spacing scale.

**Gotcha (documented):** custom properties inherit as _computed_ values. Overriding `--tss-space` on some descendant element does **not** recompute `--tss-space-4` there, because `--tss-space-4` was computed on the shell root. Overrides take full effect at the theme scope (in theme CSS), which is where themes write them.

## 4. The class catalogue

**Conventions:**

- Flat, hyphenated names: `tss-<area>[-<part>]` (`tss-nav-link`, `tss-test-status`). No BEM `__`/`--`.
- **States and variants use ARIA or data attributes, not modifier classes (D84):** `aria-current="page"` on the active nav link, `aria-expanded` on collapsible groups, `aria-pressed` on the active theme or scheme option, `aria-current="location"` on the heading a route targets (instead of `:target`, H9), `data-status="failed"` on a test status, `data-kind="hook"` on a unit badge, `data-scheme="dark"` on a showcase stage (H19). Attribute names and allowed values are catalogue constants (`tssData`, §8), and the catalogue docs have a **States and variants** section per class with a CSS example for each state.
- Every class is documented with: what element it goes on, which properties the theme styles, which the module sets (§2), and what the module may assume.

**Initial catalogue** (classes are added additively later):

| Area             | Classes                                                                                                                                                                                                                  |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Shell            | `tss-shell` (the shell root inside the shadow root; carries `data-theme`), `tss-header`, `tss-brand`, `tss-nav`, `tss-nav-group`, `tss-nav-link`, `tss-main`, `tss-footer`, `tss-breadcrumbs`                           |
| Page             | `tss-page`, `tss-page-header`, `tss-title`, `tss-subtitle`, `tss-section`, `tss-section-title`                                                                                                                         |
| README content   | `tss-content` (the container the module puts README HTML in; the theme styles the bare elements inside it as prose)                                                                                                    |
| Content elements | `tss-h1` … `tss-h6`, `tss-paragraph`, `tss-link`, `tss-list`, `tss-list-item`, `tss-code`, `tss-code-block`, `tss-blockquote`, `tss-table`, `tss-rule`, `tss-heading-anchor`                                            |
| Plain-text docs  | `tss-text-doc` (the escaped `<pre>` for `Doc.format: 'text'`, D71)                                                                                                                                                      |
| Syntax tokens    | `tss-token-<token>` for the 16 tokens of §3.3 (C5 maps its highlighter's token types onto them, D39)                                                                                                                    |
| Unit info        | `tss-badge` (`data-kind`), `tss-symbol-list`, `tss-symbol`                                                                                                                                                             |
| Showcase         | `tss-showcase`, `tss-showcase-header`, `tss-showcase-title`, `tss-showcase-stage` (holds the slot, §1; `data-scheme`), `tss-showcase-scheme` (the per-showcase light/dark selector; its options reuse `tss-theme-option`), `tss-showcase-error`, `tss-showcase-loading` |
| Tests            | `tss-tests`, `tss-test-summary`, `tss-test-file`, `tss-test-suite`, `tss-test`, `tss-test-status` (`data-status`), `tss-test-duration`, `tss-test-error`                                                              |
| Theme switcher   | `tss-theme-switcher`, `tss-theme-option`                                                                                                                                                                               |
| Utilities        | `tss-muted`, `tss-visually-hidden`                                                                                                                                                                                      |

**A content element and its class share one rule,** so README prose and module UI that opts into the same look can't drift apart:

```css
.tss-paragraph,
.tss-content p {
  margin-block: 0 var(--tss-paragraph-spacing);
  line-height: var(--tss-line-height-normal);
}
```

## 5. Scoping transform (D65, D82)

Theme CSS is written **unscoped**: variables on `:root`, class rules plain (`.tss-title { … }`, `.tss-content a { … }`); `@font-face`, `@keyframes`, `@media`, `@supports` and `@container` are allowed. The CLI turns all configured themes into **two stylesheets**:

- **the shell stylesheet**, which the module adopts into the shell's shadow root: every theme's rules, wrapped per theme;
- **the fonts stylesheet**, which the module adds to the document: the themes' `@font-face` rules, because `@font-face` declared inside a shadow root is ignored by browsers.

```css
/* theme source (unscoped) */
:root {
  --tss-color-primary: oklch(55% 0.18 255);
}
.tss-title { font-size: var(--tss-heading-1-size); }
.tss-content a { color: var(--tss-link); }
@media (prefers-reduced-motion: reduce) {
  :root { --tss-duration: 0s; }
}
```

```css
/* shell stylesheet, theme "ocean" (scheme: light) */
.tss-shell[data-theme='ocean'] {
  color-scheme: light;
}
:where(.tss-shell[data-theme='ocean']) {
  & { --tss-color-primary: oklch(55% 0.18 255); }
  .tss-title { font-size: var(--tss-heading-1-size); }
  .tss-content a { color: var(--tss-link); }
  @media (prefers-reduced-motion: reduce) {
    & { --tss-duration: 0s; }
  }
}
```

| Input                                                                                 | Transform                                                                                                                                                                                                                                       |
| ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| the whole file                                                                        | wrapped in `:where(.tss-shell[data-theme="<name>"]) { … }`; every selector inside stays exactly as written. `:where()` gives the wrapper **zero** specificity (G12), so theme rules keep their own specificity and module CSS (later in the shadow root) can win by order |
| `:root` (alone, also inside conditional rules)                                        | becomes `&` (the shell root itself)                                                                                                                                                                                                             |
| a leading `.tss-shell` compound (`.tss-shell`, `.tss-shell .tss-nav`, `.tss-shell > header`) | the `.tss-shell` part becomes `&`, so it means the shell root rather than a non-existent descendant (H17)                                                                                                                                    |
| `@media`, `@supports`, `@container`                                                   | stay in place (conditional rules can be nested)                                                                                                                                                                                                |
| `@keyframes`                                                                          | hoisted to the top level of the shell stylesheet together with any enclosing `@media`/`@supports` (they work inside shadow roots); names use the theme's own prefix (§2.1 rule 2)                                                                |
| `@starting-style`                                                                     | stays in place (nested)                                                                                                                                                                                                                         |
| `@font-face`                                                                          | moved to the **fonts stylesheet**; `font-family` names use the theme's own prefix                                                                                                                                                               |
| `@import`                                                                             | **inlined** before scoping, with the imported file's `url()`s rebased (G5); relative paths only (e.g. `../node_modules/…` is not supported; compose by file list instead); `supports()`/media conditions wrap the inlined content; `layer()` is rejected (H20) |
| `@layer`, `@property`, `@scope`, `@charset`, `@namespace`, `@page`, `@counter-style`, `@font-feature-values`, `@font-palette-values`, any other at-rule | **rejected** with an error naming the file (no layers, D81; registered properties, counter styles and font feature/palette values are document-global and would leak to showcases; the rest has no place in theme CSS) |
| `[data-theme]`, `:host`, `:host()`, `:host-context()`, `html`, `body`, `slot`, `::slotted()`, `::part()`, compound `:root` (`:root.x`, `:root .x`); `color-scheme` declarations | **rejected** (§2.1 rule 1)                                                                                                                                                      |
| added                                                                                 | a per-theme `.tss-shell[data-theme="<name>"] { color-scheme: <scheme>; }` block, always emitted, from `ThemeDefinition.scheme` (G4)                                                                                                            |
| `url(…)`                                                                              | rewritten relative to the output stylesheet's location so it still points at the original **source** asset; `data:`, absolute (`http(s):`) and `#fragment` URLs are left alone (G7). The files are toolchain inputs: the module emits the assets and rewrites the URLs relative to where the CSS ends up (or passes a matching `baseURL` to constructed sheets), since adopted sheets resolve URLs against the document (H8) |

- **Several themes sharing a file** (both default themes include `catalogue.css`): the file is emitted **once**, wrapped in `:where(.tss-shell:is([data-theme="default-light"], [data-theme="default-dark"]))`. Order is preserved by a **topological merge** of every theme's file list (file identity = realpath); if two themes order shared files inconsistently, the conflicting files are emitted per theme instead of shared (G4).
- **Name clashes:** two themes defining the same `@keyframes` or `@font-face` name with different content is an error (G6).
- **Signature:** `scopeThemeCss(themes, { shellOutFile, fontsOutFile }) → { shell: string; fonts: string }`, a pure function on PostCSS, in the CLI package (implemented in C3, wired up in C11). Test vectors cover every row above, shared-file grouping, an order conflict, a theme made only of shared files, a composed theme, inlined `@import`s (with conditions), a leading `.tss-shell` compound, a font/image asset `url()`, and every rejection.
- **Handing over (D82):** C11 writes both files atomically (temp file + rename) to the CLI's cache directory (`node_modules/.cache/tsshowcase/`, owned by C11) and passes their absolute paths to the module as `BuildContext.themeStylesheets` (`{ shell, fonts }`). **In `dev`** the CLI regenerates them in place and also **pushes the new contents** through `DevContext.onThemeStylesheetsUpdate(listener)` (H7: Vite doesn't watch `node_modules`); `-base-vite` forwards them over its live channel and calls `replaceSync()` on the adopted sheets. The dev server must be allowed to read the cache and theme asset directories (`server.fs.allow`, C15).
- **Browser support:** the output keeps CSS nesting (decided, H18): Baseline 2023, with bare type selectors in nested blocks needing the relaxed syntax (Chrome 120, Safari 17.2, late 2023), fine for a documentation site; a toolchain that lowers nesting for older targets (Vite's default `cssTarget` in builds) produces equivalent flat CSS. Shadow DOM, slots and constructable stylesheets are Baseline (Safari 16.4 last). No `@scope` dependency.

## 6. No cascade layers (D81)

No `@layer` for theme or module CSS. Isolation comes from the shadow root (§1), not from the cascade; inside the shadow root, theme rules (zero-specificity wrapper) come first and module CSS after them.

## 7. `-theme-default` (the two default themes)

```
packages/tsshowcase-theme-default/
  css/
    tokens.css      derived scales + component tokens (formulas only)
    light.css       base token values + semantic picks for light
    dark.css        base token values + semantic picks for dark
    catalogue.css   every catalogue class and README element style, using variables only; ends with the reduced-motion block
  src/
    DefaultThemes/      the ThemeSet class (default export of the package) + README + tests
    defaultThemeFiles/  named absolute paths { tokens, light, dark, catalogue } for composing themes (G20) + README
    index.ts
```

- `default-light` = `[tokens.css, light.css, catalogue.css]`; `default-dark` = `[tokens.css, dark.css, catalogue.css]`. Paths resolved with `fileURLToPath(new URL('../../css/…', import.meta.url))` (absolute filesystem paths, as `ThemeDefinition.css` requires, H25), which works from `src/` and `dist/` alike; `files: ["dist", "css"]`; an `exports` entry `"./css/*": "./css/*"` lets composing themes reference the files by package path.
- **Fonts:** system font stacks only (`system-ui`, `ui-monospace`, …): no `@font-face`, nothing fetched (C2 rule 2).
- **Accessibility:** visible focus rings (`--tss-focus-ring`, `--tss-focus-width`) on every interactive catalogue class; reduced motion (§3.2); **contrast**: WCAG AA for text on surfaces, muted text, links, accent text, code and every status colour on its surface, a review guideline in C3 and checked by a computed-contrast smoke test on the built output in C12's browser run (D85, H31).
- **Composition stays as in C2 §7:** a custom theme takes `[...base.css, mine.css]` (or picks named files from `defaultThemeFiles`) and overrides base or semantic tokens.
- The package README is for users theming their showcase (D83): the two themes, values at a glance, the composition pattern, which tokens to override for common goals.

## 8. The catalogue in `-api` (fills the C2 slot)

One source of truth, `catalogue/catalogue.ts`, an `as const` metadata object:

```ts
export const catalogue = {
  variables: {
    colorPrimary: { name: '--tss-color-primary', layer: 'base', type: 'color', description: 'Primary brand colour; drives the primary ramp, accent and links' },
    colorPrimary300: { name: '--tss-color-primary-300', layer: 'derived', type: 'color', description: '…' },
    // …
  },
  classes: {
    shell: { name: 'tss-shell', area: 'shell', description: 'The shell root inside the shadow root; carries data-theme', module: ['display', 'grid-template-*'], theme: ['background', 'color', 'font-*'] },
    // …
  },
  data: {
    status: { name: 'data-status', values: ['passed', 'failed', 'skipped', 'todo', 'running', 'queued', 'unavailable', 'not-run'], description: '…' },
    kind: { name: 'data-kind', values: ['component', 'hook', 'function', 'class', 'constant', 'type'], open: true, description: 'Well-known unit kinds; modules may add their own' },
    expanded: { name: 'aria-expanded', values: ['true', 'false'], description: '…' },
    pressed: { name: 'aria-pressed', values: ['true', 'false'], description: '…' },
    current: { name: 'aria-current', values: ['page', 'location'], description: 'page: the active nav link; location: the heading a route targets' },
    scheme: { name: 'data-scheme', values: ['light', 'dark'], description: 'The selected scheme of a showcase stage' },
  },
} as const;
```

- **Generated constants (G18):** `npm run catalogue` generates literal `tssVar`, `tssClass` and `tssData` constants from the metadata (`ci:catalogue` checks nothing drifted), so browser bundles import plain literals and the metadata tree-shakes away. Mapped types derive their types. `cssVar()` stays as in C2.
- **Generated docs (D83):** the same script generates the reference into the `catalogue/` unit README between markers: every variable (layer, type, description), every class (area, description, module vs theme properties), every data/ARIA attribute (values), plus the layer diagram, the naming convention, the §3.5 gotcha and the **States and variants** section.

## 9. Enforcement (D46, D88)

One policy, two linters, shipped as the published **`@ofzza/tsshowcase-lint`**. **Enforced** on our own theme and module packages (`-theme-default`, `-base-react`, `-module-react-default`, including our dogfood showcases, D41); **opt-in** for third-party theme, module and runner authors; never applied to a showcased user project.

### 9.1 Stylelint rules (`tsshowcase/…`)

- **`design-values`:** banned in declarations: colour literals (hex, `rgb()`/`hsl()`/`oklch()`…, named colours other than `transparent`/`currentColor`), dimensions with units other than `0`, and bare numbers other than `0`/`1` outside the structural allow-list. Allowed: `var(--tss-…)` (and own-prefix variables, below), percentages, structural keywords, inside `calc()`/`color-mix()` unitless multipliers and percentages combined with variables, and the **structural allow-list (G8):** `fr` units (`1fr`, `minmax(…, 1fr)`), `100vh`/`100vw`/`100dvh`/`100svh`, ratio values in `aspect-ratio`, `deg` in multiples of 90 inside `rotate()`, numbers in `flex`/`flex-grow`/`flex-shrink`/`order`/`grid-row`/`grid-column`, `color-scheme` values (`light`, `dark`, `normal`) and the `all` keywords (`initial`, `inherit`, `unset`, `revert`), which the module's host and slot resets need (H24).
  - **Literal exception (G17):** literals are allowed only in declarations of catalogue variables whose layer is `base` (theme packages) and of the theme's own-prefix variables.
- **`known-variables`:** every `--tss-…` variable used or declared must be a catalogue variable. Other variables only with an allowed prefix (option `ownPrefixes`); our packages configure none.
- **`scoping`:** in theme CSS, no `[data-theme]`, `:host`, `:host()`, `:host-context()`, `html`, `body`, `slot`, `::slotted()`, `::part()` selectors, no compound `:root`, no `color-scheme` declarations (§2.1 rule 1); `overflow`/`transform`/`filter`/`contain`/`container-type`/`will-change` on the documented stage-ancestor classes are reported (§1 layout rules).
- **`own-prefix`:** a theme's own classes, variables, `@keyframes` names and `font-family` names must not use `tss-` unless they are catalogue names (§2.1 rule 2).
- **Escape hatch:** `/* stylelint-disable-next-line tsshowcase/design-values -- <reason> */`, with `reportDescriptionlessDisables: true`.

### 9.2 ESLint rules

- **`tsshowcase/style-values`:** the same value policy for JSX `style={{ … }}` and CSS-in-JS object literals; string values must be variables, `calc()`/`color-mix()` of them, percentages or structural keywords; numeric values only `0` and the numeric allow-list mirroring stylelint's (`flex`, `flexGrow`, `flexShrink`, `order`, `gridRow`, `gridColumn`); template literals only when every expression is a `tssVar`/`cssVar()` reference.
- **`tsshowcase/known-classes`:** `tss-…` class literals must be catalogue classes (constants preferred: `tssClass.title`); in module code every class must be a catalogue class or match the module's `ownPrefixes` (arrangement-only classes, §2.1 rule 3).
- **Escape hatch:** `// eslint-disable-next-line tsshowcase/style-values -- <reason>`, with `eslint-comments/require-description`.

### 9.3 The `@ofzza/tsshowcase-lint` package (D88)

A **published** package of lint rules, plugins and presets. It contains no code that runs linting: each consumer runs stylelint/ESLint with its own config files.

| Export                    | Kind   | For                                                                                                                                     |
| ------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| `stylelintPlugin`         | plugin | the four `tsshowcase/…` stylelint rules (§9.1)                                                                                          |
| `eslintPlugin`            | plugin | `tsshowcase/style-values`, `tsshowcase/known-classes` (§9.2)                                                                            |
| `stylelintConfigs.theme`  | preset | theme CSS: `design-values` (base-token literal exception), `known-variables`, `scoping`, `own-prefix`, descriptionless disables reported |
| `stylelintConfigs.module` | preset | module CSS: `design-values` with **no** literal exception, `known-variables`                                                            |
| `eslintConfigs.module`    | preset | module TSX/JS: `style-values`, `known-classes`, `eslint-comments/require-description`                                                   |
| runners                   | –      | no styling rules apply (runners render nothing); the README says so                                                                     |

- **Dependencies (G11):** `stylelint` (≥ 16) and `eslint` (≥ 9) are peer dependencies; **`@ofzza/tsshowcase-api` is a peer dependency** too, so the rules check against exactly the catalogue the author's plugin uses; `@eslint-community/eslint-plugin-eslint-comments`, `postcss-value-parser`, `postcss-selector-parser` and `@typescript-eslint/utils` are dependencies.
- **Versioning:** like every package (`0.1.0`, changesets, D80); a rule change that rejects previously accepted code is breaking (a minor bump before 1.0).
- **Docs (user-facing):** the README is written for theme, module and runner authors: which preset for which kind of plugin, setup, each rule's purpose (linking each rule's README and D46/D86), `ownPrefixes`, the escape hatch.
- **Added by C3** (C1 unchanged; D88). C3 also carries out the "adds a package" duties of C1 §10a (G22): AGENTS layout and packages list, the root `tsconfig.build.json` reference, a `unit:tsshowcase-lint` Vitest project, the dependency table (including the CLI's new `postcss` dependency for the transform, H40), CONTRIBUTE structure, the root `stylelint.config.js` plus stylelint devDependencies, and `ci:stylelint` / `ci:catalogue` under C1's script wiring rule.
- **Gotcha (G22, G39):** the root lint configs import the **built** `-lint` (Node resolves `dist`; the `source` condition is for TypeScript only), so lint needs a build first. `ci:build` runs before the lint steps and `prepare` builds on install; after `npm run clean`, run `npm run build` before linting. Recorded in AGENTS.md gotchas.

## 10. Runtime theme switching (what C3 specifies, what C15/C10 implement)

- **Selection and persistence:** C2 inclusion rules 4–5 (the algorithm and constants), implemented in `-base-vite` (C15). The shell is rendered by JavaScript into the shadow root, so the theme attribute is set before the shell first paints (no flash of the wrong theme).
- **The theme switcher:** `tss-theme-switcher` with one `tss-theme-option` per theme (`aria-pressed` on the active one), rendered inside the shadow root (C10). Hidden when only one theme is configured.
- **Switching** = setting `data-theme` on `.tss-shell`; everything in the shell restyles at once.
- **Per-showcase scheme selector (H12):** `tss-showcase-scheme`, following the active theme's `scheme` until changed manually; a manual choice holds until the next theme switch, which resets every showcase to the new theme's scheme; not persisted. Changing it re-renders the showcase with the new `colorScheme` and updates the slot's `color-scheme` and the stage's `data-scheme` (C10).
- **Live OS changes:** the initial selection reads `prefers-color-scheme` once at start; later OS changes are ignored in v1 (H16).
- **The page itself:** on every theme switch the module updates `color-scheme` and the canvas background on `<html>` (§1, H6).

## 11. Documentation

- **Audiences (D83):** the `-api` `catalogue/` README is for people **writing** a theme or a module; the `-theme-default` README is for users **theming their showcase**; the `-lint` README is for plugin authors opting into our lint presets.
- `catalogue/` README (generated parts, §8) plus **"Building a theme that works with any module"** (D86): the isolation model in one paragraph (§1: why a theme only ever styles the shell), the three rules of §2.1 each with its reason, and recommendations: start by composing `-theme-default`, override base tokens before semantic ones, keep WCAG AA contrast, use the `stylelintConfigs.theme` preset. C13's theme authoring guide expands it.
- AGENTS.md: the trigger row "added or renamed a catalogue name → `npm run catalogue`, update both default themes"; the gotchas (§3.5, reduced motion last, lint needs a build) and the D89 pitfalls (typography cut off at the slot, document rules vs `:host`, `rem` follows the user's root, clipping and stacking under shell ancestors, fragments and `:target` don't reach the shadow root, Vite doesn't watch `node_modules`).
- `CONTRIBUTE.md` glossary: base token, derived scale, semantic token, component token, theme scope element, shell (shadow root), slotted showcase.

## 12. Tests

All Node-side (no browser tests, G10):

- **Catalogue (`-api`):** names prefixed and unique; every variable has a layer, type and description; the generated constants match the metadata (`ci:catalogue`); type tests for the derived types.
- **Default themes (`-theme-default`):** **completeness** (parsing the CSS, every catalogue variable is declared by both themes through their files, and no undeclared `--tss-` variable is used); **layering** (only `base` variables carry literals); the reduced-motion block is in the last file.
- **Scoping transform (CLI):** the vectors listed in §5.
- **D89 behaviour** is verified in C12's browser run (a fixture with a global reset and a portal component, per-showcase scheme switching, README heading links, theme fonts, dark scrollbars, computed contrast); a throwaway browser spike is recommended while planning C10/C15 (H27).
- **Lint rules (`-lint`):** stylelint and ESLint rule tests (valid/invalid cases, the structural allow-list, `ownPrefixes`, the escape hatch with and without a reason).
- **Dogfood:** `ci:stylelint` and ESLint pass on the theme and module packages.

## Acceptance criteria

1. `npm run ci` passes, including `ci:stylelint` and `ci:catalogue`.
2. The catalogue metadata is the only place names are written; the generated constants and README list every name.
3. Both default themes declare every catalogue variable and pass the layering test.
4. The scoping transform's vectors pass (§5), including shared files emitted once, an order conflict, inlined imports, a leading `.tss-shell` compound, `@font-face` in the fonts stylesheet and every rejection.
5. A theme file with a literal colour in a class rule, an unknown `--tss-` variable, a `[data-theme]`/`:host`/`body`/`::slotted()` selector, a `color-scheme` declaration, or an own name with the `tss-` prefix fails `stylelintConfigs.theme`; the same with a `-- reason` disable passes. Module CSS using `1fr`, `100dvh` and `rotate(90deg)` passes `stylelintConfigs.module`.
6. An inline `style={{ color: 'red' }}` fails `eslintConfigs.module`; `style={{ color: cssVar(tssVar.text) }}` and `style={{ flex: 1 }}` pass.

**Review guidelines** (checked by hand in C3, G10): overriding only `--tss-color-primary` in a composed theme visibly changes the accent, links and active nav; the default themes meet WCAG AA contrast (also smoke-tested in C12, D85).

## Changes to approved plans (applied to the documents, re-approved with this plan)

- **C2, amendment B1 (revised, round 6 additions from `03b`):** rule 3/7 split into binding vs free (H34); the `themeStylesheets` hand-off described (contents, assets, dev updates via `DevContext.onThemeStylesheetsUpdate`, H7/H8/H35); rule 4: the document-level `color-scheme`/canvas exception and no live OS following (H6, H16); overlays inside `.tss-shell` (H29); the showcase wrapper and showcase stylesheet option (H1); `themes` without raw `css` (H37); heading ids per doc (H36); references fixed (H30). Earlier: fills the catalogue slot (§8: `tssVar`, `tssClass`, `tssData`, generated); `BuildContext.themeStylesheets: { shell, fonts }` (D82); inclusion rule 3 (adopt the shell stylesheet into the shell's shadow root first, add the fonts stylesheet to the document, `data-theme` on `.tss-shell`), rule 6 (README HTML in `tss-content`, inside the shell), rule 7 (showcases isolated from the shell's styles with portals working, receiving a colour scheme through a module-defined interface, per-showcase scheme selector, whole-showcase opt-in to render inside the shell; D89); §7 theme rules (§2.1); the composition sample's CSS path (`../css/ocean.css`, G33); the dev-watching note (G27); `resolveRoute` prose (G35).
- **Super-plan, revision R2 (revised, round 6 additions):** C15's runtime API and C10's duties (H28); C12's D89 acceptance and contrast smoke test (H33, H31); the `-base-vite`/`-base-react` rows (H30). Earlier: the `-lint` row and the updated `-theme-default` and CLI rows (G41); C3 owns the isolation model's specification, the scoping transform and the lint package; C10 implements the shadow-root shell, slotted showcases, scheme selectors and the shell opt-in; C11 owns the CLI cache, atomic writes and calls the transform (edge C11 → C3, G34); C15 adopts the shell stylesheet into the shadow root, adds the fonts stylesheet, allows the cache and asset directories, and hot-reloads adopted stylesheets; C5 uses `tssData`/token classes; C13 references `-lint`; C14 publishes `-lint` after `-api`; stale C2/C13 text fixed (G28); C1's package list excludes `-lint` (G30).
- **AGENTS.md:** D89 reworded (no "exactly as in the user's app" overclaim, H41) and the §6 pitfalls (H42); D85 (C12 contrast smoke test); D7 wording (H44); D89 (isolation model); D44, D81, D86, D87, D85, D82, D88 revised accordingly; D35/D42 wording (G42); resume steps list the review files (G45).
- **C1:** no change.

## Out of scope

The shell/showcase rendering, scheme selector and switcher components (C10), adopting stylesheets and hot-reloading them (C15), syntax-highlighter integration (C5), the module's layout CSS (C10), authoring guides (C13).

## Decisions (all resolved)

- ✅ **Q1 What a theme styles (D86, revised by D89):** anything inside the shell, extending the catalogue if needed, within the three rules of §2.1.
- ✅ **Q2 Cascade layers (D81):** none.
- ✅ **Q3 Who scopes (D82):** the CLI, into a shell stylesheet and a fonts stylesheet.
- ✅ **Q4 Lint tooling (D88):** the published `@ofzza/tsshowcase-lint`.
- ✅ **Q5 Catalogue docs (D83):** generated into the `-api` catalogue README.
- ✅ **Q6 States and variants (D84):** ARIA/data attributes.
- ✅ **Q7 Contrast (D85, revised):** a review guideline in C3, smoke-tested in C12's browser run; no helper.
- ✅ **Isolation (D89):** Shadow-DOM shell, light-DOM slotted showcases, colour scheme passed to showcases; supersedes regions, `@scope` and `tss-plain` (D87).
- ✅ **Round 6 (`03b` group B):** showcase wrapper + showcase stylesheet option (H1); keep `rem` (H3); dev stylesheet push (H7); scheme selector behaviour (H12); keep nesting (H18); contrast smoke test in C12 (H31).
