---
pr: kubernetes-sigs/prow#694
title: "spyglass/junit: add configurable test grouping support"
head_sha: d9a45d7d93042060c7f11ac22b90fa79215d7ed2
base: main
reviewed_at: 2026-06-30T16:56:09Z
verdict: approve
---

## Findings

### [should-fix] Empty shadow-box table when all tests match groups
- where: `pkg/spyglass/lenses/junit/template.html:90-145`
- concern: When groups are configured and every test matches a group, the default bucket is empty. The outer `<table class="mdl-shadow--2dp">` at line 90 is emitted unconditionally with no rows. The first group closes it at line 145. The `mdl-shadow--2dp` class renders a visible empty box with drop shadow above group content.
- excerpt: |
    <table id="junit-table" class="mdl-data-table mdl-js-data-table mdl-shadow--2dp">
    {{/* default sections all guarded by {{if gt $numX 0}} -- all skipped */}}
    {{range $gix, $group := .Groups}}
    {{if gt $group.NumTests 0}}
    </table>

### [nit] Group matching uses only tests[0] without documenting assumption
- where: `pkg/spyglass/lenses/junit/lens.go:460`
- concern: Converging concern (Code Quality + Maintainability). Group selector checks only `tests[0].Result`. Correct because all entries share the same testIdentifier and properties are test-definition metadata, but assumption is undocumented.
- excerpt: |
    if compiled[i].predicate.matches(tests[0].Result) {

### [nit] Comment says "interface" but type is struct
- where: `pkg/spyglass/lenses/junit/lens.go:272`
- concern: Comment reads "testBucket is an interface" but the type is `struct`.
- excerpt: |
    // testBucket is an interface for appending classified test results.
    type testBucket struct {

### [nit] Table close/reopen trick is subtle and uncommented
- where: `pkg/spyglass/lenses/junit/template.html:145-147`
- concern: Each non-empty group closes current table and opens a new one to insert an h5 header. Guarded by NumTests > 0 but structurally fragile. Deserves a comment.
- excerpt: |
    </table>
    <h5 class="group-header">{{$group.Name}}</h5>
    <table class="mdl-data-table mdl-js-data-table mdl-shadow--2dp">

### [question] Template duplication for passed/skipped sections
- where: `pkg/spyglass/lenses/junit/template.html:170-203`
- concern: Second commit extracted test-detail-rows sub-template for failed/flaky. Passed and skipped sections still duplicated between default and group blocks. Lower severity (simpler markup) but still a sync burden.

### [question] Multi-predicate selectors supported but undocumented
- where: `pkg/spyglass/lenses/junit/lens.go:120-134`
- concern: Parser handles multiple properties/property[] segments in a single selector (AND semantics). Docs and examples only show single-predicate selectors.

## Checked
- classifyTests() is faithful refactoring of old inline classification logic
- NumTests accounting via prevTotal/newTotal delta is equivalent to old formula
- TotalTests() used only for empty-check; per-section fractions use bucket-local counts
- getJvd variadic signature works for both call sites
- Nil/empty testGroups to classifyTests is a no-op
- Empty-predicate propertyPredicate cannot be constructed through normal parsing
- CSS/TS changes correctly extend existing patterns to group-* classes
- Test coverage: TestParseSelector, TestSelectorMatches, TestGetJvdWithGroups, two template cases
- Documentation accurate with worked examples
- No security concerns: regex parsing, html/template auto-escaping, operator config only
- Fully backward compatible, zero behavioral change without groups config
- Deployment risk LOW: opt-in, no migration, rollback safe, zero overhead when unconfigured

## Open questions
- Empty-table fix preference: gate default table on {{if gt .NumTests 0}}, or restructure to avoid close/reopen?
- Interest in extracting passed/skipped into shared sub-templates (here or follow-up)?
- Multi-predicate AND semantics worth documenting?

## Followups

### cleanup (should): Fix empty table visual artifact + edge-case test
- where: `pkg/spyglass/lenses/junit/template.html:17-183`, `lens_test.go`
- necessity: should
- prompt: |
    ```
    In kubernetes-sigs/prow, following PR #694 ("spyglass/junit: add configurable
    test grouping support", merge commit 8f72e9c5), fix a visual artifact in the
    JUnit lens template and add test coverage for the edge case.

    ## Problem

    In `pkg/spyglass/lenses/junit/template.html`, the outer
    `<table id="junit-table" class="mdl-data-table mdl-js-data-table mdl-shadow--2dp">`
    is emitted unconditionally at line 17. When groups are configured and every test
    matches a group (default bucket is empty), this table has zero content rows. The
    first group's `</table>` at line 183 closes it, producing an empty styled box
    with a drop shadow above the actual group content.

    ## Task

    1. In `template.html`, gate the default `<table id="junit-table">` so it is
       only emitted when the default bucket has tests (`.NumTests > 0`). When only
       groups have tests, each group should emit its own standalone `<table>` without
       an empty predecessor. Make sure the closing `</table>` at the end of the
       template also respects this gating.

    2. In `pkg/spyglass/lenses/junit/lens_test.go`, add a test case to
       `TestGetJvdWithGroups` where ALL tests have properties matching the configured
       group, so `expFailed` and `expPassed` for the default bucket are both 0 and
       all tests appear in the group.

    3. Add a corresponding template test case (in the `TestTemplate` function) that
       renders a `JVD` with `NumTests: 0` and all tests in a `GroupResult`, and
       verify the HTML output does NOT contain an empty `<table` before the group
       header (i.e., no `<table` tag immediately followed by `</table>`).

    ## Acceptance criteria
    - `go test ./pkg/spyglass/lenses/junit/...` passes
    - The new test case covers the all-tests-in-groups scenario
    - The template produces clean HTML (no empty styled table) when the default
      bucket is empty

    ## Out of scope
    - Template duplication cleanup (separate followup)
    - Changes to lens.go logic (only template and tests change)
    ```

### cleanup (should): Unify failed/flaky template rendering into shared sub-templates
- where: `pkg/spyglass/lenses/junit/template.html`, `junit.css`, `lens.ts`
- necessity: should
- prompt: |
    ```
    In kubernetes-sigs/prow, following PR #694 ("spyglass/junit: add configurable
    test grouping support", merge commit 8f72e9c5), reduce template duplication in
    the JUnit lens.

    ## Problem

    `pkg/spyglass/lenses/junit/template.html` has four near-identical copies of the
    "expandable test result row" markup:
    - Default failed section (lines ~24-93)
    - Default flaky section (lines ~102-145)
    - Group failed section (lines ~196-265)
    - Group flaky section (lines ~274-317)

    They differ only in CSS class names: `failure-layout`/`failure-name`/`failure-text`
    vs `flaky-layout`/`flaky-name`/`flaky-text` vs `group-layout`/`group-name`/
    `group-text`. Any rendering change must be replicated in all four places.

    ## Task

    1. Unify the CSS classes. The failed, flaky, and group sections use different
       class names (`failure-*`, `flaky-*`, `group-*`) but identical styling (check
       `junit.css` — the selectors are comma-separated). Replace these with a single
       set of class names (e.g., `detail-layout`, `detail-name`, `detail-text`) used
       by all three section types. Update `junit.css` to remove the now-redundant
       duplicate selectors. Update `lens.ts` if it references any of the old class
       names.

    2. Extract the test-detail row rendering (both single-run and multi-run
       variants) into a named sub-template using Go's `{{define "test-detail-rows"}}`
       / `{{template "test-detail-rows" .}}`. Both default and group sections should
       invoke this same sub-template, passing their respective test result slice.

    3. The passed and skipped sections are simpler (no expand/collapse for individual
       tests) — leave them inline; they don't carry enough duplication to justify
       extraction.

    ## Acceptance criteria
    - `go test ./pkg/spyglass/lenses/junit/...` passes — all existing template tests
      must produce equivalent output (the HTML structure is the same, only class
      names change)
    - The template file is significantly shorter (roughly halved for the
      failed/flaky sections)
    - `junit.css` has no redundant comma-separated selectors for the old class names
    - `lens.ts` still correctly handles expand/collapse for all sections

    ## Out of scope
    - Changes to lens.go logic or test classification
    - Changes to the passed/skipped section rendering
    - New features or behavioral changes
    ```
