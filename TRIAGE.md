---
issue: kubernetes-sigs/prow#915
title: "branchprotector cannot manage required status check GitHub App identity"
state: open
labels: []
main_sha: ffc57790ba3e94ab2081a4a5d498d8baba5db8f1
triaged_at: 2026-09-07T18:54:32Z
verdict: accepted
refresh_log:
  - previous_triaged_at: 2026-09-04T11:25:42Z
    summary: "Incorporated @weshayutin's non-substantive '@myprc fyi' comment; state and labels unchanged, with no new cross-reference."
---

## Findings

### [reproducibility] Same context name can be rejected for App identity
- detail: The reporter provides a PR where GitHub rejected a successful `Run CI` check because it was not produced by the App GitHub expected for the required check.
- evidence: `https://github.com/openshift/velero-plugin-for-gcp/pull/165`; GitHub branch-protection REST API supports `checks[].app_id` and `-1` for any App.

### [cause] Prow models only legacy context names
- detail: The shared state/request type has no representation for GitHub's `required_status_checks.checks[].app_id`; branchprotector cannot send or read App identity.
- where: `pkg/github/types.go:686-690`
- excerpt: |
    type RequiredStatusChecks struct {
        Strict   bool     `json:"strict"`
        Contexts []string `json:"contexts"`
    }

### [cause] Request construction emits only contexts
- detail: Desired checks are deduplicated by name and serialized as `contexts`, retaining GitHub's implicit recent-App selection behavior.
- where: `cmd/branchprotector/request.go:63-70`
- excerpt: |
    return &github.RequiredStatusChecks{
        Contexts: append([]string{}, sets.List(sets.New[string](cp.Contexts...))...),
        Strict:   makeBool(cp.Strict),
    }

### [cause] Reconciliation ignores producer identity
- detail: A returned required check with the desired context name but a different App ID is treated as equal, so branchprotector does not repair it.
- where: `cmd/branchprotector/protect.go:575-584`
- excerpt: |
    return state.Strict == request.Strict &&
        equalStringSlices(&state.Contexts, &request.Contexts)

### [related-code] Policy inheritance is context-name-only
- where: `pkg/config/branch_protection.go:75-83`, `pkg/config/branch_protection.go:148-162`, `pkg/config/branch_protection.go:472-489`
- detail: `ContextPolicy` and its merge function union string contexts; automatically required presubmits are injected through that same field.

### [related-code] Existing full branch-protection update can carry checks
- where: `pkg/github/client.go:2906-2917`
- detail: Branchprotector PUTs its `BranchProtectionRequest` to GitHub's full branch-protection endpoint. GitHub accepts `required_status_checks.checks` there; no different API call is required.

### [related-issue] Required-context reconciliation
- ref: kubernetes-sigs/prow#841
- relevance: Covers a different required-context reconciliation failure; it does not support App identities.

### [related-issue] GitHub Apps in branch restrictions
- ref: kubernetes/test-infra#24530
- relevance: Reportedly added App support for push/merge restrictions, a different branch-protection subresource.

## Checked
- Issue #915 is open with no labels or comments; its stated SHA equals the triage worktree's HEAD, `ffc57790ba3e94ab2081a4a5d498d8baba5db8f1`.
- No existing branchprotector configuration field or documentation supports a required-check `app_id`.
- GitHub's official REST documentation confirms `required_status_checks.checks[].app_id`; `-1` explicitly permits any App.
- An equality-only fix is insufficient because desired state cannot currently preserve or request App identity.
- Existing tests cover legacy policy merging, request construction, and context-only equality, but not App identity.

## Next steps
- Apply `kind/feature` and `area/branchprotector`; consider `help-wanted` after agreeing on the schema.
- Add an additive explicit-check policy, e.g. `checks: [{context: Run CI, app_id: -1}]`, while preserving existing `contexts` behavior if `checks` is absent.
- Define validation and precedence for overlap with legacy contexts and auto-added presubmit contexts; merge explicit entries by context and reject conflicting identities.
- Extend GitHub wire types, request rendering, and equality normalization to compare deterministic `(context, app_id)` pairs.
- Add config/merge, serialization, mismatch, `-1`, legacy compatibility, and documentation coverage.

## Open questions
- Should overlapping `contexts` and `checks` be rejected, or have documented precedence?
- If `checks[].app_id` is omitted, should Prow omit it and let GitHub select, or require an explicit numeric ID or `-1`?
- Can an operator override the identity of an automatically managed Prow presubmit context, or must those retain legacy behavior?
