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
