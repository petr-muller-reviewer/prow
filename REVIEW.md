---
pr: kubernetes-sigs/prow#942
title: "feat(deck): add dark mode support for main prow status dashboard"
head_sha: d37116e6bc583ddfadff0288b649db55fe91b2af
base: main
reviewed_at: 2026-09-23T12:15:41Z
verdict: request-changes
---

## Verdict

Request changes. The index-only dark-mode behavior is appropriately scoped, but storage-restricted browsers can leave the toggle uninitialized, and `!important` dark-mode rules override explicitly configured Deck branding colors.

## What this PR does

- Adds early theme selection on the main Deck dashboard from a saved browser preference or the OS color scheme.
- Adds an index-page header control that persists an explicit light/dark choice.
- Switches the default logo with the theme while leaving custom logos alone.
- Adds index-scoped dark styling for the dashboard's navigation, filters, table, histogram, and dialogs.

## Findings

### [blocking] Theme initialization assumes browser storage is always available
- where: `cmd/deck/template/base.html:40-45`, `cmd/deck/template/base.html:143-160`, `cmd/deck/template/base.html:166-169`
- concern: `localStorage.getItem` and `setItem` may throw when storage is disabled by browser privacy policy or embedding restrictions. An exception at line 144 aborts `initTheme` before the click handler is registered, so users cannot toggle the theme; later writes can also prevent the UI update. Wrap storage access in helpers and fall back to the system preference plus an in-memory choice for the current page.
- excerpt: |
    var stored = localStorage.getItem('prow-theme');
    ...
    localStorage.setItem('prow-theme', nextTheme);

### [blocking] Dark mode supersedes operator-configured Deck branding
- where: `cmd/deck/static/style.css:1152-1161`; `cmd/deck/template/base.html:53-56`
- concern: Deck renders configured background and header colors inline, but the new dark-mode rules force different colors with `!important`. This silently changes an operator's branding for dark-preferring visitors and can leave a retained custom logo with inadequate contrast. Preserve explicitly configured colors in dark mode, or add and document an intentional configurable dark-branding policy.
- excerpt: |
    <body id="{{.PageName}}"{{if branding.BackgroundColor}} style="background-color: {{branding.BackgroundColor}};"{{end}}>
    <header class="mdl-layout__header"{{if branding.HeaderColor}} style="background-color: {{branding.HeaderColor}};"{{end}}>
    ...
    background: #121212 !important;
    ...
    background-color: #212121 !important;

### [should-fix] Icon-only theme control has no accessible name
- where: `cmd/deck/template/base.html:63-65`
- concern: The button has a visual `title`, but no stable accessible name; its icon text changes with state. Add an `aria-label` and update it alongside the title so screen-reader users are told what action the control will take.
- excerpt: |
    <button id="theme-toggle" class="mdl-button mdl-js-button mdl-button--icon" title="Toggle dark mode">
      <i id="theme-toggle-icon" class="material-icons">brightness_2</i>
    </button>

### [should-fix] Theme state and palette add avoidable maintenance surface
- where: `cmd/deck/static/style.css:1152-1410`; `cmd/deck/template/base.html:103-185`
- concern: Every new dark selector is duplicated for `html.dark-mode body#index` and `body#index.dark-mode`, and repeated literal palette values are spread through the stylesheet. The controller is also inline in the shared base template despite being index-specific. Use one root class, semantic CSS variables for repeated colors, and move post-DOM behavior into the existing index frontend bundle; keep only the minimal FOUC bootstrap inline.
- excerpt: |
    html.dark-mode body#index,
    body#index.dark-mode {
      background: #121212 !important;
    }

## Checked

- The change is limited to `PageName == "index"`; other Deck pages retain their existing behavior.
- The head bootstrap applies the root class before body rendering, minimizing a light-theme flash.
- An explicit stored choice takes precedence over the system preference, and system changes are followed only without one.
- Custom logo URLs are retained rather than replaced by the default dark/light assets.
- No Prow API, config schema, RBAC, dependency, or server-side availability behavior changes.
- Existing `deckVersion` URLs continue to cache-bust changed static resources; no special upgrade sequencing is needed.

## Open questions

- Should a configured `branding.background_color` or `branding.header_color` always win, or should Deck grow explicit dark-branding configuration?
- Is the project prepared to support browsers/environments where `localStorage` is unavailable, with non-persistent theme switching as the fallback?

## Followups

Post-merge opportunities identified 2026-11-13 against PR #942 (merged as `e62c14122b7de539fb60236fb8085c0fd1f5dc83`). Five accepted, one skipped (docs subsection on `site/content/en/docs/components/core/deck/_index.md`). Each prompt is self-contained and targets current `main`.

### deferred-review: Harden theme preference storage when `localStorage` is unavailable
- where: `cmd/deck/template/base.html:40,144,159,167`
- necessity: must
- mode: deliver
- why: Unfixed blocking finding "Theme initialization assumes browser storage is always available"; a throw aborts `initTheme` before the toggle listener is attached.

```text
In kubernetes-sigs/prow, following PR #942 — "feat(deck): add dark mode support for main prow status dashboard" (merged as e62c14122b7de539fb60236fb8085c0fd1f5dc83) — harden Deck's client-side theme persistence in cmd/deck/template/base.html.

The index-page bootstrap (lines ~38-47) and the initTheme script (lines ~103-176) call localStorage.getItem/setItem directly at lines 40, 144, 159, and 167. Browsers that block storage throw, which aborts initTheme before the click handler is attached and leaves the toggle dead; a failed setItem also skips updateThemeUI.

Task: introduce safeStorageGet(key) / safeStorageSet(key, value) helpers that wrap localStorage access in try/catch and fall back to an in-memory variable for the page lifetime. Route every localStorage access in this template through them. Ensure the toggle click listener is registered even when storage is unavailable, and that a failed write still calls updateThemeUI so the theme changes for the current page. Keep all other behavior identical.

Acceptance criteria: with localStorage throwing on both get and set (stub globalThis.localStorage to throw), the page still resolves a theme from prefers-color-scheme, the toggle flips theme/icon/title/logo for the current page, and the console shows no uncaught exception. With storage available, persisted-preference behavior is unchanged.

Scope guard: do not refactor the duplicated CSS selectors, move the controller into prow.ts, touch branding rules, add aria-label, or change Go code.
```

### deferred-review: Preserve operator-configured branding in dark mode
- where: `cmd/deck/static/style.css:1155-1165`; `cmd/deck/template/base.html:52-54`
- necessity: should
- mode: deliver
- why: Unfixed blocking finding "Dark mode supersedes operator-configured Deck branding"; `!important` dark rules discard configured background/header colors. Priority policy remains an open question, so this should resolve toward branding winning.

```text
In kubernetes-sigs/prow, following PR #942 — "feat(deck): add dark mode support for main prow status dashboard" (merged as e62c14122b7de539fb60236fb8085c0fd1f5dc83) — preserve operator-configured Deck branding under dark mode.

cmd/deck/template/base.html lines 52-54 render branding.BackgroundColor and branding.HeaderColor as inline styles on <body> and <header>. cmd/deck/static/style.css lines 1155-1165 force background #121212 and header background #212121 with !important under html.dark-mode body#index, silently overriding them.

Task: make explicitly configured branding win over dark-mode defaults. Emit data attributes (for example data-branded-background="true" and data-branded-header="true") in base.html when the corresponding branding value is set, and scope the dark-mode body/header background rules so they do not apply to those attributes (or use another equivalent mechanism that lets the inline branding color win). Leave the rest of the dark palette unchanged.

Acceptance criteria: with deck.branding.background_color and header_color configured, dark mode shows exactly those colors on body and header while the rest of the index page stays dark; with no branding configured, dark mode is visually identical to today. No !important dark rule overrides an inline branding color.

Scope guard: do not add new config fields or a dark-branding policy, change logo selection, add accessibility labels, or modify unrelated CSS. If you conclude branding should not win in dark mode, stop and report instead of inventing policy.
```

### deferred-review: Give the theme toggle an accessible name
- where: `cmd/deck/template/base.html:63-65`; title updates near `base.html:133-140`
- necessity: should
- mode: deliver
- why: Unfixed should-fix finding "Icon-only theme control has no accessible name"; the button has only a `title` and a changing icon glyph.

```text
In kubernetes-sigs/prow, following PR #942 — "feat(deck): add dark mode support for main prow status dashboard" (merged as e62c14122b7de539fb60236fb8085c0fd1f5dc83) — give the index-page theme toggle a stable accessible name.

The button at cmd/deck/template/base.html:63-65 has only title="Toggle dark mode" and an icon whose glyph changes; the controller updates toggleBtn.title around lines 135 and 161, but no aria-label exists.

Task: add aria-label to the button, initialized to "Switch to dark mode" to match the initial action, and update it in updateThemeUI alongside toggleBtn.title so it always describes what clicking will do. Keep the existing icon and title behavior.

Acceptance criteria: the rendered button exposes a stable accessible name in both themes, changing between "Switch to dark mode" and "Switch to light mode" in sync with the title. No visual change.

Scope guard: only cmd/deck/template/base.html; do not restructure the controller or touch CSS.
```

### tech-debt: Deduplicate dark selectors and move the theme controller into `prow.ts`
- where: `cmd/deck/static/style.css:1152-1413`; `cmd/deck/template/base.html:103-176`; `cmd/deck/template/index.html:4`
- necessity: should
- mode: deliver
- why: Unfixed should-fix finding "Theme state and palette add avoidable maintenance surface"; every dark selector is duplicated and index-only JS lives in the shared base template.

```text
In kubernetes-sigs/prow, following PR #942 — "feat(deck): add dark mode support for main prow status dashboard" (merged as e62c14122b7de539fb60236fb8085c0fd1f5dc83) — reduce duplication in the new dark-mode implementation.

cmd/deck/static/style.css lines 1152-1413 duplicate every selector for html.dark-mode body#index and body#index.dark-mode and repeat literal palette values. cmd/deck/template/base.html lines 103-176 hold roughly 70 lines of index-only controller inline in the shared template, while the index page already loads cmd/deck/static/prow/prow.ts as prow_bundle.min.js (cmd/deck/template/index.html:4).

Task: standardize on a single root state class (prefer html.dark-mode, applied by the existing pre-body bootstrap), rewrite the dark rules once against that class plus body#index, and extract the repeated colors (#121212, #212121, #1e1e1e, #2d2d2d, #e0e0e0, #ffffff) into semantic CSS custom properties on the dark root. Move the post-DOM controller (toggle wiring, matchMedia listener, logo swap) from base.html into cmd/deck/static/prow/prow.ts, keeping only the minimal FOUC bootstrap inline in the template. Make sure the prow_bundle.min.js build path and Bazel/rollup configuration still work.

Acceptance criteria: no duplicated dark selector pairs remain; the dark palette is defined once in variables; toggle, persistence, OS-preference following, and logo swap behave exactly as before on index; other Deck pages are unaffected; the Go build and the existing frontend build target succeed.

Scope guard: index page only. Do not change the visual design, branding behavior, accessibility labels, or the static .DarkMode setting, and do not add features. Coordinate with issue #944 but do not implement its cross-page or logo-architecture work here.
```

### tech-debt: Retire the static `.DarkMode` setting and finish theming consolidation (issue #944)
- where: `cmd/deck/templates.go:32,37-38,71-72`; `cmd/deck/template/base.html:47-50`; `cmd/deck/template/spyglass.html:53`
- necessity: could
- mode: deliver
- why: Deck now has two theming mechanisms; issue #944 documents the dual-track problem and a consolidation plan, and the merge makes it worth executing.

```text
In kubernetes-sigs/prow, following PR #942 — "feat(deck): add dark mode support for main prow status dashboard" (merged as e62c14122b7de539fb60236fb8085c0fd1f5dc83) — implement issue #944 (deck: consolidate static .DarkMode template setting with dynamic client-side dark mode).

Deck passes baseTemplateSettings.DarkMode (cmd/deck/templates.go:32, constructor at 37-38, helpers darkMode/lightMode at 71-72) solely to choose the default logo in cmd/deck/template/base.html:47-50. spyglass.html:53 passes darkMode while every other page passes lightMode, and the new client-side .dark-mode theme overlaps with it.

Task: follow the issue's plan incrementally on current main. (1) Remove DarkMode from baseTemplateSettings and delete the darkMode/lightMode helpers, updating every template call site. (2) Switch the default logo light/dark choice to a CSS/picture-based mechanism that works for both statically dark pages (spyglass) and dynamic theming, while still honoring custom branding logos. (3) Decide and document whether Spyglass stays forced-dark and whether other Deck pages adopt dynamic theming; implement the low-risk part now and record any larger rollout on #944. Split into small commits or PRs if needed.

Acceptance criteria: no baseTemplateSettings.DarkMode or darkMode/lightMode helpers remain; every Deck page renders the correct logo for its intended theme; index dynamic theming still works; go build ./cmd/deck/... and template rendering tests pass; any deferral is documented on #944.

Scope guard: do not redesign Deck styling, change the dark palette, or alter branding color semantics; do not duplicate the selector-dedup followup. Link the work to #944.
```
