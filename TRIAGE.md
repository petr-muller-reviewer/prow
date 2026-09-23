---
issue: kubernetes-sigs/prow#944
title: "deck: consolidate static `.DarkMode` template setting with dynamic client-side dark mode"
state: open
labels: 
main_sha: 695a33027101f624c2bbd80f6943b47abb46d09e
triaged_at: 2026-09-23T12:55:26Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/feature, area/deck, help-wanted]
---

# Triage

## Verdict

Accepted. This is a well-scoped Deck presentation cleanup: legacy server-side logo selection now overlaps with the index dashboard’s browser-side dark-mode preference introduced by PR #942. It is actionable after one product decision: whether Spyglass remains permanently dark or participates in a future dynamic theme.

## What the issue reports

- Deck retains a static `.DarkMode` setting while the index dashboard now tracks theme in `localStorage` and `prefers-color-scheme`.
- The shared base template chooses a logo on the server, while index JavaScript can choose the same logo again in the browser.
- Spyglass alone passes the static dark setting and is styled dark independently.
- The requested direction is one theming model, removal of obsolete template state, and non-branching logo presentation.

## Findings

### [reproducibility] Index theme selection has two sources of truth

- detail: The server renders the index with the light default logo, then the browser may select dark based on a persisted preference or system setting and swaps that logo.
- evidence: `cmd/deck/template/index.html:90`; `cmd/deck/template/base.html:34-44,100-183`.

### [cause] `DarkMode` is legacy logo-selection state

- detail: The field was introduced for the 2019 logo change and is not a general Deck theme system. It is the sole server-side input to the base template’s default-logo choice.
- evidence: `cmd/deck/templates.go:30-38,64-75`; `cmd/deck/template/base.html:46-49`.

### [related-code] Only Spyglass uses the static dark argument

- where: `cmd/deck/template/spyglass.html:53`
- excerpt: |
    {{template "page" (settings mobileUnfriendly darkMode "spyglass" .)}}
- relevance: The other ten `settings` callers pass `lightMode`, so removing the boolean is mechanically contained but needs a deliberate Spyglass initial-logo decision.

### [related-code] Dynamic theme support is intentionally index-only

- where: `cmd/deck/static/style.css:1150-1417`
- excerpt: |
    html.dark-mode body#index,
    body#index.dark-mode {
        background: #121212 !important;
    }
- relevance: Non-index Deck pages share the base template but do not have corresponding dark-mode styles or a toggle.

### [related-code] Spyglass has an independent hard-coded dark palette

- where: `cmd/deck/static/spyglass/spyglass.css:24-30`
- relevance: Dynamic-theme removal must not inadvertently make the existing Spyglass presentation visually inconsistent.

### [related-pr] Dynamic index dark mode

- ref: kubernetes-sigs/prow#942
- relevance: Merged 2026-09-17 as `d37116e6b`; it changed only `cmd/deck/template/base.html` and `cmd/deck/static/style.css` and created the overlapping client-side mechanism.

## Checked

- Issue #944 is open, has no labels or comments, and concerns Prow-owned Deck code.
- `baseTemplateSettings.DarkMode` is consumed only by `base.html`; `darkMode` and `lightMode` are registered only to supply that argument.
- Eleven templates call `settings`; Spyglass is the sole `darkMode` caller.
- Searched Deck tests and documentation: no test covers template rendering, logo selection, stored/system theme preference, or toggle behavior; no user-facing dark-mode documentation/configuration exists.
- The maintainer completed the mandatory seven-slide briefing: validity, cause, technical scope, recommendation, and effort assessment were confirmed.

## Next steps

- Apply `kind/feature`, `area/deck`, and `help-wanted` manually if the issue is to be advertised to contributors.
- Clarify whether Spyglass is an intentional fixed-dark page or should participate in a dynamic theme.
- For the narrow change, remove `DarkMode`, the two boolean template functions, and the boolean `settings` argument from all callers; preserve custom `branding.Logo` behavior.
- Add focused rendered-template coverage for default and custom logos, plus browser/DOM coverage for index persisted and system-preference paths.
- Keep full dark-mode support for non-index pages as separately scoped work unless maintainers explicitly choose it now.

## Open questions

- “Should Spyglass remain intentionally dark, with a fixed logo presentation, or should it join a future user-selectable Deck theme?”
