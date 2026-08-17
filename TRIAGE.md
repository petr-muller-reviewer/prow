---
issue: kubernetes-sigs/prow#477
title: "branchprotector: excluded branches retain existing protection instead of being removed"
state: closed
labels: lifecycle/rotten
main_sha: da30d8b3fec2bdb5c5e923f85df17faa04520d6f
triaged_at: 2026-06-15T16:21:44Z
verdict: needs-discussion
---

## Findings

### [cause] `exclude` skips branches entirely — semantics are ambiguous
- detail: In `UpdateRepo`, excluded branches hit `continue` at line 341-343 and are never added to the `branches` map. They never reach `UpdateBranch`, so no action is taken — neither applying nor removing protection. This makes `exclude` behave identically to `unmanaged: true` (don't touch), not like `protect: false` (actively remove). Whether this is a bug depends on the intended semantics of `exclude`, which the documentation does not clarify.
- evidence: `cmd/branchprotector/protect.go:341-343`

### [related-code] Branch enumeration and filtering
- where: `cmd/branchprotector/protect.go:328-347`
- excerpt: |
    } else if !ok && branchExclusions != nil && branchExclusions.MatchString(b.Name) {
        logrus.Infof("%s/%s=%s: excluded", orgName, repoName, b.Name)
        continue
    }

### [related-code] configureBranches — removal dispatch
- where: `cmd/branchprotector/protect.go:182-196`
- excerpt: |
    if u.Request == nil {
        if err := p.client.RemoveBranchProtection(u.Org, u.Repo, u.Branch); err != nil {
            p.errors.add(fmt.Errorf("remove %s/%s=%s protection failed: %w", u.Org, u.Repo, u.Branch, err))
        }
        continue
    }

### [related-code] UpdateBranch — where protect:false triggers removal
- where: `cmd/branchprotector/protect.go:454-517`
- excerpt: |
    var req *github.BranchProtectionRequest
    if *bp.Protect {
        r := makeRequest(*bp, p.enableAppsRestrictions)
        req = &r
    }
    // If protect is false, req stays nil -> triggers RemoveBranchProtection

### [related-code] Exclude field definition
- where: `pkg/config/branch_protection.go:54-56`
- excerpt: |
    // Exclude specifies a set of regular expressions which identify branches
    // that should be excluded from the protection policy, mutually exclusive with Include
    Exclude []string `json:"exclude,omitempty"`

### [related-code] Existing exclusion tests — gap: no excluded+protected case
- where: `cmd/branchprotector/protect_test.go:1098-1167`
- excerpt: Tests verify excluded branches are not protected. No test checks behavior when an excluded branch is already protected on GitHub.

### [related-pr] PR #478 — proposed fix
- ref: kubernetes-sigs/prow#478
- relevance: By the same author (kaovilai). Adds a second pass after branch enumeration to detect excluded-but-protected branches and queue removal requests. Has `lgtm` label, all CI green, waiting for `approve`. Changes `exclude` semantics from "don't manage" to "actively remove," which is a behavior change. Bypasses `equalBranchProtections` dedup, causing extra API calls for each excluded protected branch.

## Checked
- `Exclude` field documented as "branches that should be excluded from the protection policy" — ambiguous
- `Unmanaged` flag takes a different code path (per-branch policy property vs. enumeration-level filter) but has the same "don't touch" effect
- `Include` filtering has the same gap (branches that stop matching an include pattern keep existing protection)
- Existing tests only verify excluded branches don't receive protection, not that existing protection is removed
- Fake test client's `GetBranches` correctly simulates `Protected=true` for `onlyProtected=true` call
- Three distinct semantics exist: `protect: false` (remove), `unmanaged: true` (don't touch, per-branch), `exclude` (don't touch, pattern-based) — issue author wants `exclude` to match `protect: false`
- Current `exclude` behavior has been stable for years

## Next steps
- Reopen issue #477 (closed by stale bot, not by maintainer decision)
- Resolve design question: should `exclude` mean "don't manage" (current, like `unmanaged`) or "remove protection" (like `protect: false`)?
- If "remove protection": review PR #478 for backwards compatibility (deployments using `exclude` to leave manually-managed branches alone) and API consumption (removal calls bypass `equalBranchProtections` dedup)
- If "don't manage": close as working-as-designed, suggest author use per-branch `protect: false`, improve docs to clarify `exclude` vs `unmanaged`
- Apply label: `area/branchprotector`

## Open questions
- What are the intended semantics of `exclude`? Documentation is ambiguous.
- If `exclude` should remove protection: is there a backwards-compatible way to introduce this?
- Should `include` handle removal when a branch stops matching the pattern? Same class of ambiguity.
- PR #478 bypasses `equalBranchProtections` dedup — API call burst concern for repos with many excluded protected branches.
- Historical tension between boolean config (`protect`/`unmanaged`) and pattern config (`include`/`exclude`): should docs differentiate more clearly?
