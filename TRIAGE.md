---
issue: kubernetes-sigs/prow#1096
title: "Spyglass HTML lens: srcdoc escaping doesn't escape `&`, breaks nested iframes and entities"
state: open
labels: kind/bug, area/deck, area/spyglass
main_sha: dc7498690949cec828864910cebfc4aeef2453ad
triaged_at: 2026-11-26T18:26:07Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 1
recommended_labels: [kind/bug, area/spyglass]
---

# Triage

## Verdict

**accepted** — legitimate, well-scoped rendering bug in the Spyglass HTML lens, with a confirmed fix already in flight as PR #1097.

The report names the exact code, cites the relevant WHATWG spec section, supplies a concrete job URL that renders blank, and proposes a stdlib fix plus a test. It is actionable as written and belongs in this repository.

## What the issue reports

- The HTML lens embeds every artifact as `<iframe srcdoc="{{.Content}}">`.
- Before embedding, only `"` is escaped to `&quot;`; `&` is left untouched.
- Because a browser decodes the attribute value and then parses the srcdoc document, pre-existing references are decoded one stage too early.
- Result: `&lt;b&gt;` becomes real `<b>` markup, and a nested `srcdoc` containing `&quot;` is closed early, so nested previews render blank.
- The reporter proposes `html.EscapeString(doc.Content)` and a test asserting a single srcdoc decode returns the original bytes.

## Findings

### [reproducibility] blank nested preview in a real job
- detail: The linked job's `output/report.html` is blank inside Spyglass but renders when the artifact is opened directly. The page uses nested iframes whose `srcdoc` values contain entities.
- evidence: https://prow.ci.openshift.org/view/gs/test-platform-results-public/pr-logs/pull/openshift-eng_ai-helpers/875/pull-ci-openshift-eng-ai-helpers-main-eval-runner/2125582949825908736

### [cause] srcdoc escaping omits `&`
- detail: The artifact string is escaped only for double quotes (`strings.ReplaceAll(doc.Content, "\"", "&quot;")`), leaving `&` untouched. The srcdoc attribute value is character-reference-decoded by the attribute parser, then the resulting document is parsed again; escaping `&` first prevents the extra decode.
- evidence: `pkg/spyglass/lenses/html/html.go:128`

### [related-code] escaping runs after the height-notifier injection
- where: `pkg/spyglass/lenses/html/html.go:124-128`
- excerpt: |
    doc.Content = injectHeightNotifier(doc.Content, doc.ID)
    ...
    doc.Content = strings.ReplaceAll(doc.Content, `"`, `&quot;`)

The injected `<div>` and `<script>` must also be escaped, so the fix belongs after `injectHeightNotifier`.

### [related-code] srcdoc attribute is rendered by text/template
- where: `pkg/spyglass/lenses/html/template.html:16`
- excerpt: |
    <iframe srcdoc="{{.Content}}" title="{{.Filename}}" sandbox="allow-scripts allow-popups allow-same-origin" id="{{.ID}}" width="100%" scrolling="no"></iframe>

`text/template` does no contextual escaping, so the manual escape in `html.go` is the only protection for the attribute.

### [related-code] title and description are unescaped in the outer page
- where: `pkg/spyglass/lenses/html/html.go:131`, `pkg/spyglass/lenses/html/template.html:10`
- excerpt: |
    doc.Title = doc.Title + fmt.Sprintf(` <abbr class="icon material-icons" title="%s">info</abbr>`, description)
    <h6>{{.Title}}</h6>

Same `text/template` page renders artifact-derived `Title` and meta `description` without escaping. Out of scope for #1096; flagged in Open questions as a possible follow-up.

### [related-pr] PR #1097 implements the fix
- ref: kubernetes-sigs/prow#1097
- relevance: Open, non-draft, `lgtm` + `approved`, labeled `area/spyglass`; switches to `html.EscapeString`, adds a `With character references` fixture, and says `Fixes #1096`. Unit test and lint pass; `pull-prow-image-build-test` and `pull-prow-integration` are retesting (`mergeStateStatus: UNSTABLE`).

## Checked

- Keyword searches `spyglass srcdoc`, `html lens escape`, and PR search `srcdoc escape` — no duplicate issue or competing PR; only #1096 and #1097.
- `git log` / `git blame` on `pkg/spyglass/lenses/html/` — manual `"` escaping dates to 2022 (228dd527d); height-notifier injection was added later (0b7c78f70).
- Baseline `go test ./pkg/spyglass/lenses/html/... -run TestRenderBody` on `main` (dc74986) passes.
- PR #1097 diff and its new fixture reviewed: expected output is `&amp;lt;` / `&amp;quot;`, confirming the intended double-escape.
- No other `srcdoc` or `EscapeString` sites exist in the HTML lens.
- `TestRenderBody` fixtures cover `"` but not pre-existing entities or nested `srcdoc`; PR #1097 adds the entity/nested case.

## Next steps

- Merge PR #1097 once `pull-prow-image-build-test` and `pull-prow-integration` go green; it is already `lgtm` + `approved`.
- Optionally ask the PR author to add the decode-once round-trip assertion the issue describes, instead of relying only on a snapshot fixture.
- Keep `kind/bug` and `area/spyglass`; do not add `good-first-issue` because PR #1097 already exists.
- Consider whether `area/deck` is warranted on the issue; the owning code is `area/spyglass`, which the PR uses.

## Open questions

- Separate follow-up: artifact `<title>` text and `<meta name="description">` content are inserted into the outer page unescaped. A description containing a decoded `"` could break out of its `title="..."` attribute. Should this be filed as its own issue after #1097 lands?
- For the PR: would you add a test that decodes the srcdoc attribute once and asserts it equals the escaped original, rather than snapshotting only?
