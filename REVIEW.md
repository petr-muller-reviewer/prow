---
pr: kubernetes-sigs/prow#733
title: "auxiliary extra_refs"
head_sha: 5f29693a9de3c261031aee93a1079edd8415317d
base: main
reviewed_at: 2026-09-28T21:51:32Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-06-15T16:00:42Z
  gated_head_sha: 5f29693a9de3c261031aee93a1079edd8415317d
  reviewed_head_sha: 5f29693a9de3c261031aee93a1079edd8415317d
refresh_log:
  - from_sha: 5f29693a9de3c261031aee93a1079edd8415317d
    to_sha: 5f29693a9de3c261031aee93a1079edd8415317d
    at: 2026-06-02T23:41:37Z
    summary: "No code change. One /cc comment from petr-muller (2026-06-01). No new reviews or inline comments."
  - from_sha: 5f29693a9de3c261031aee93a1079edd8415317d
    to_sha: 5f29693a9de3c261031aee93a1079edd8415317d
    at: 2026-09-28T21:51:32Z
    summary: "No code change. petr-muller approved; bots added lgtm and approved labels; PR merged 2026-06-15."
---

## Gate

**Decision: MERGE**

No code changes since the review. All findings were nits and questions — none blocking. @petr-muller approved on 2026-06-15; the `lgtm` and `approved` labels followed, and the PR merged at 2026-06-15T17:17:26Z. The author (@pohly) left one inline comment on `types.go:1236` acknowledging that `Auxiliary` may also affect pod labeling but explicitly deferring that to a future PR.

**Area 1 — Prior findings disposition:** All six findings from REVIEW.md are nits/questions. None were addressed in code (SHA unchanged), but none are blocking. The author's inline comment about pod labeling is a new scope note, not a concern about this PR's correctness.

**Area 2 — Independent merge risk:**
- **API surface:** Adds `Auxiliary bool` to the exported `Refs` struct — purely additive, no removals or renames. `json:"auxiliary,omitempty"` means no serialization change for existing objects.
- **CRD:** New optional boolean field, no validation webhook changes, no defaulting changes. Existing CRs unaffected.
- **Behavior:** Only changes version selection in `started.json` for jobs that explicitly opt in via `auxiliary: true`. No behavioral change for any existing job.
- **Blast radius:** None for existing deployments. The field is inert until set.

**Gating list:** Nothing gates. All findings are follow-up quality.

**Process gate outcome:** At gate time, the PR still needed `lgtm` and an OWNERS approval. @petr-muller's approval satisfied that requirement; k8s-ci-robot added `lgtm` and `approved` on 2026-06-15, before merging the PR.

## Summary

Adds `Auxiliary bool` to `Refs` struct. Periodic jobs with `extra_refs` used only for tooling (e.g. kops, test-infra) can mark those refs so version selection for `started.json` skips them. Core change: `mainRefs()` loops instead of blindly returning `extra[0]`. CRD updated. Backward-compatible (bool zero value = old behavior). Three independent reviewer perspectives all reached APPROVE; two converging concerns worth addressing as follow-ups.

Since previous review (2026-05-28T23:19:12Z):
- No code changes; SHA unchanged.
- petr-muller posted `/cc` on 2026-06-01 (reviewer assignment bot command).
- No new reviews or inline comments.

Since previous refresh (2026-06-02T23:41:37Z):
- No code changes; head remains `5f29693a9de3c261031aee93a1079edd8415317d`.
- petr-muller approved with "LGTM" on 2026-06-15T16:00:07Z; k8s-ci-robot posted an approval notice and added `lgtm` and `approved` labels at 16:03Z.
- The PR merged on 2026-06-15T17:17:26Z. No new inline comments.

## Findings

### [nit] All-auxiliary edge case untested and silent
- where: `pkg/pod-utils/downwardapi/jobspec.go:232-241`
- concern: When all ExtraRefs have `Auxiliary: true` and `ProwJobSpec.Refs` is nil, `mainRefs()` returns nil → `GetRevisionFromSpec` returns `""` → `started.json` gets an empty version string. No test covers this path, no log warns about it. Flagged by all three reviewer perspectives independently.
- excerpt: |
    for i := range extra {
        if !extra[i].Auxiliary {
            return &extra[i]
        }
    }
    return nil

### [nit] Missing SpecToStarted-level test for auxiliary skipping
- where: `pkg/pod-utils/downwardapi/jobspec_test.go`
- concern: The feature's stated purpose is what ends up in `started.json`, but tests only cover `GetRevisionFromSpec`. The `refsToStarted` path (via `SpecToStarted`/`PjToStarted`) calls `mainRefs` directly and is not tested with an auxiliary first extra_ref and non-auxiliary second extra_ref.

### [nit] Auxiliary on primary Refs is silently a no-op
- where: `pkg/apis/prowjobs/v1/types.go:1225-1236` and `pkg/pod-utils/downwardapi/jobspec.go:232-234`
- concern: `mainRefs()` short-circuits on `refs != nil` before checking `Auxiliary`, so setting `auxiliary: true` on `pj.spec.refs` does nothing. The field doc doesn't clarify this. An operator who sets it on the primary `refs:` block expecting version-selection behavior would be silently wrong.
- excerpt: |
    func mainRefs(refs *prowapi.Refs, extra []prowapi.Refs) *prowapi.Refs {
        if refs != nil {
            return refs   // Auxiliary on primary refs never checked
        }

### [nit] "Refs from many extra_refs" test name is ambiguous
- where: `pkg/pod-utils/downwardapi/jobspec_test.go:327-338`
- concern: The test validates pre-existing behavior (first of multiple non-auxiliary extra_refs is chosen), not new behavior. The name doesn't signal this is a regression guard. A one-line comment would clarify intent for future readers.

### [question] Config validation for Auxiliary on primary refs
- where: `pkg/config/` (no change in this PR)
- concern: No validation warns when `auxiliary: true` is set on `ProwJobSpec.Refs` directly. Could be a follow-up validation in the config loader.

### [question] GCS reporter path not tested with auxiliary refs
- where: `pkg/crier/reporters/gcs/reporter.go:118`
- concern: Calls `GetRevisionFromRefs(pj.Spec.Refs, pj.Spec.ExtraRefs)` directly. Logic is covered via the shared function, but no test exercises this call site with auxiliary refs.

## Checked

- `shaForRefs` uses `reflect.DeepEqual` on full `Refs` struct; `Auxiliary` will be consistent between spec and clone record — no phantom mismatch.
- `refsToStarted` still includes auxiliary refs in `started.Repos` map (lines 281-283) — correct, those repos were cloned; only version selection is affected.
- Both CRD YAML locations updated (list schema and top-level refs schema), content matches `types.go` comment.
- `mainRefs` is called from both `GetRevisionFromSpec` and `refsToStarted`; both paths correctly skip auxiliary refs.
- Loop uses `&extra[i]` not a copy — avoids the address-of-loop-variable bug.
- `omitempty` ensures backward compatibility; existing configs round-trip identically.
- `bool` field (not pointer) requires no changes to generated DeepCopy code.
- CRD schema updated in sync with Go type in the same PR.
- Field only affects version reporting, not cloning — no risk of skipping required checkouts.

## Open questions

- When all extra_refs are auxiliary and no primary Refs exist, is an empty version string in `started.json` acceptable (kubetest2 always overwrites via metadata.json in those cases), or should there be a log warning?
- Should `Auxiliary: true` on `ProwJobSpec.Refs` be explicitly rejected at config validation time, or is a doc comment clarification sufficient?

## Followups

Followup pass (2026-10-10T16:31:16Z): retained the four previously accepted
handoff prompts below. Accepted no additional followups; skipped three:

- Additional helper edge-case tests and clarification of the regression-test name.
- Investigation of validation for `Auxiliary` on primary `Refs`; retain the documentation clarification.
- GCS reporter boundary tests for auxiliary refs and absent primary refs.

The PR description confirms that an empty version when all refs are auxiliary
is intentional: it allows TestGrid to use the tester-provided metadata version.
The retained started.json tests should preserve that behavior.

### docs: Clarify that Auxiliary is a no-op on primary Refs
- category: docs
- necessity: should
- where: `pkg/apis/prowjobs/v1/types.go:1225-1236`, CRD YAML (two locations)
- prompt:

```
In kubernetes-sigs/prow, following merged PR #733 ("auxiliary extra_refs",
merge commit 351e8cfd58915657bd36a50e7e86bbe972bc0739), the new
`Auxiliary` bool field on the `Refs` struct has a doc comment that says "the
first repository where Auxiliary is false or unset is considered the main
repository and determines the version." This is misleading: `mainRefs()` in
`pkg/pod-utils/downwardapi/jobspec.go` unconditionally returns
`ProwJobSpec.Refs` when non-nil, before checking `Auxiliary`. Setting
`auxiliary: true` on the primary `spec.refs` does nothing.

Task: Add one sentence to the `Auxiliary` field's Go doc comment in
`pkg/apis/prowjobs/v1/types.go` (around line 1225) clarifying: "This field
has no effect when set on ProwJobSpec.Refs; it is only meaningful on elements
of ExtraRefs." Then mirror the same sentence into the two CRD YAML description
blocks in `config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml`
(search for "Auxiliary indicates" — it appears twice, once under `extra_refs`
items and once under the top-level `refs`).

Acceptance criteria: the three description texts (Go comment, two CRD YAML
blocks) all contain the clarification. Run `go build ./...` to confirm the
change compiles. No other files should be modified.

Out of scope: adding config validation to reject `Auxiliary: true` on primary
refs, adding tests, changing behavior.
```

### docs: Mention auxiliary field in jobs.md
- category: docs
- necessity: should
- where: `site/content/en/docs/jobs.md:48`
- prompt:

```
In kubernetes-sigs/prow, following merged PR #733 ("auxiliary extra_refs",
merge commit 351e8cfd58915657bd36a50e7e86bbe972bc0739), a new
`auxiliary: true` field can be set on `extra_refs` entries to tell Prow that
the repo is only providing tooling/helper files and should be skipped when
determining the version for started.json. The job configuration guide at
`site/content/en/docs/jobs.md` shows a periodic job example with `extra_refs`
(around line 48) but doesn't mention this field.

Task: In the periodic job YAML example in `jobs.md`, add a second `extra_refs`
entry with `auxiliary: true` set, and add a short inline YAML comment
(e.g. `# auxiliary: not used for version in started.json`) or a single
sentence immediately after the code block noting that `auxiliary: true` tells
Prow to skip the ref for version selection. Keep it minimal — one line of
explanation, not a paragraph.

Acceptance criteria: the example shows a concrete use of `auxiliary: true`
and a reader can understand what it does without leaving the page. No other
docs files should be modified.

Out of scope: creating a new docs section, adding detailed explanation of
TestGrid version semantics, modifying any Go code.
```

### tests: Cover started.json version selection with auxiliary refs
- category: tests
- necessity: should
- where: `pkg/initupload/run_test.go:29`, `pkg/pod-utils/downwardapi/jobspec.go:251-285`
- why: The merged PR tests revision selection directly, but not the `started.json` output that uses the selected ref and clone records.
- prompt:

```
In kubernetes-sigs/prow, following merged PR #733 ("auxiliary extra_refs",
merge commit 351e8cfd58915657bd36a50e7e86bbe972bc0739), extend
TestSpecToStarted in pkg/initupload/run_test.go. The PR changed mainRefs in
pkg/pod-utils/downwardapi/jobspec.go to skip auxiliary extra refs when
selecting the repository version for started.json. Current tests cover
GetRevisionFromSpec and older SpecToStarted cases, but no SpecToStarted case
contains an auxiliary ref.

Add table cases for a periodic JobSpec with an auxiliary first ExtraRefs
entry and a non-auxiliary second entry, both with and without clone records.
Assert that RepoCommit and DeprecatedRepoVersion come from the second ref
(its final clone SHA when a matching record is supplied), while Repos still
contains both cloned repositories. Add an all-auxiliary ExtraRefs case with
no primary Refs and assert the current empty version and complete Repos map.

Acceptance criteria: the new cases pass with `go test ./pkg/initupload` and
would fail if version selection reverted to ExtraRefs[0]. Keep the existing
version behavior; this task records it rather than deciding whether an empty
all-auxiliary version needs a warning.

Out of scope: changing production version selection or GCS reporter logic.
```

### deferred-review: Align pod repository labels with auxiliary refs
- category: deferred-review
- necessity: should
- where: `pkg/pod-utils/decorate/podspec.go:105-146`, `pkg/pod-utils/downwardapi/jobspec.go:232-241`
- why: The author deferred pod labeling in the review; an auxiliary-first periodic currently labels the pod with a different repository from the one selected for its version.
- prompt:

```
In kubernetes-sigs/prow, following merged PR #733 ("auxiliary extra_refs",
merge commit 351e8cfd58915657bd36a50e7e86bbe972bc0739), resolve the
pod-labeling question the author deferred in the PR review. PR #733 makes
pkg/pod-utils/downwardapi/jobspec.go select the first non-auxiliary extra
ref as the repository that determines the started.json version. However,
LabelsAndAnnotationsForSpec in pkg/pod-utils/decorate/podspec.go still uses
spec.ExtraRefs[0] for the pod's org, repo, and base-ref labels when spec.Refs
is nil.

Inspect consumers of these labels and document whether they identify the
tested repository or the first checkout. For a periodic job with an
auxiliary first extra ref and a later non-auxiliary ref, make the label
selection consistent with that documented meaning. Preserve primary Refs
behavior and define the all-auxiliary/no-ref result explicitly rather than
indexing into an absent main ref. Add focused tests for auxiliary-first,
all-auxiliary, and primary-Refs cases in the decorate package.

Acceptance criteria: the selected label values follow the documented rule;
the new cases pass with `go test ./pkg/pod-utils/decorate`; no code path
panics when all extra refs are auxiliary. Explain any intentionally retained
difference between repository labels and started.json version selection.

Out of scope: changing job URL prefixes, rerun authorization, or GCS default
configuration selection.
```
