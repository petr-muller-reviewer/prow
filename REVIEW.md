---
pr: kubernetes-sigs/prow#733
title: "auxiliary extra_refs"
head_sha: 5f29693a9de3c261031aee93a1079edd8415317d
base: main
reviewed_at: 2026-06-02T23:41:37Z
verdict: approve
refresh_log:
  - from_sha: 5f29693a9de3c261031aee93a1079edd8415317d
    to_sha: 5f29693a9de3c261031aee93a1079edd8415317d
    at: 2026-06-02T23:41:37Z
    summary: "No code change. One /cc comment from petr-muller (2026-06-01). No new reviews or inline comments."
---

## Summary

Adds `Auxiliary bool` to `Refs` struct. Periodic jobs with `extra_refs` used only for tooling (e.g. kops, test-infra) can mark those refs so version selection for `started.json` skips them. Core change: `mainRefs()` loops instead of blindly returning `extra[0]`. CRD updated. Backward-compatible (bool zero value = old behavior). Three independent reviewer perspectives all reached APPROVE; two converging concerns worth addressing as follow-ups.

Since previous review (2026-05-28T23:19:12Z):
- No code changes; SHA unchanged.
- petr-muller posted `/cc` on 2026-06-01 (reviewer assignment bot command).
- No new reviews or inline comments.

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
