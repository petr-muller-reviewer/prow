---
issue: kubernetes-sigs/prow#477
title: "branchprotector: excluded branches retain existing protection instead of being removed"
state: open
labels: area/branchprotector
main_sha: cced76d904946c1412da1ddc2d6441167d3d8e46
triaged_at: 2026-08-17T22:57:32Z
verdict: needs-discussion
---

## Findings

### [reproducibility] Confirmed, root-caused by reporter and independently verified
- detail: Add a branch to `exclude`, then run branchprotector against a branch that was already `Protected=true` on GitHub. Protection is never removed; branchprotector logs `excluded` and takes no further action. Subsequent pushes fail with `GH006: Protected branch update failed`.
- evidence: issue body (kaovilai, 2025-06-16); `cmd/branchprotector/protect.go:341-343`

### [cause] `exclude` is a pure enumeration-time skip — confirmed intentional by maintainer
- detail: In `UpdateRepo`, excluded branches hit `continue` at line 341-343 and are never added to the `branches` map, so they never reach `UpdateBranch` and no removal is ever queued, regardless of current protection state. Maintainer (petr-muller) confirmed 2026-08-13T12:12:31Z this is intended: `exclude` is meant as a pattern variant of `unmanaged: true`, not of `protect: false`, and its semantics will not be changed.
- evidence: `cmd/branchprotector/protect.go:341-343`; issue comment 2026-08-13T12:12:31Z

### [cause] No pattern-based mechanism exists to retroactively remove protection — the real gap
- detail: `Unmanaged` (`pkg/config/branch_protection.go:32-34`) is a per-branch-name boolean, not a regex list; `Exclude`/`Include` are the only regex mechanisms and both are pure "don't touch" filters. For branches whose names aren't knowable in advance (bot/automation-created, e.g. Copilot-assigned branches), there is no pattern-based way to pre-empt protection, and once branchprotector's default policy protects them, no pattern-based way to remove that protection afterward — only a manual, exact-name `protect: false` or manual GitHub UI/API action. Surfaced via a live exchange between kaovilai and petr-muller and is what shifted the maintainer from a firm won't-fix toward "softening" on a new additive field.
- evidence: `pkg/config/branch_protection.go:30-60`; issue comments 2026-08-13T12:45:09Z, 12:52:16Z, 12:53:28Z, 12:54:55Z, 12:56:28Z, 13:18:08Z

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

### [related-code] Policy struct — Unmanaged is boolean, not pattern-based
- where: `pkg/config/branch_protection.go:32-34`
- excerpt: |
    // Unmanaged makes us not manage the branchprotection.
    Unmanaged *bool `json:"unmanaged,omitempty"`

### [related-code] Exclude field definition — docstring still unclarified
- where: `pkg/config/branch_protection.go:54-56`
- excerpt: |
    // Exclude specifies a set of regular expressions which identify branches
    // that should be excluded from the protection policy, mutually exclusive with Include
    Exclude []string `json:"exclude,omitempty"`

### [related-code] Existing exclusion tests — gap: no excluded+protected case
- where: `cmd/branchprotector/protect_test.go:1098-1167`
- excerpt: Tests verify excluded branches are not protected. No test checks behavior when an excluded branch is already protected on GitHub.

### [related-pr] PR #478 — proposed fix, ruled out as-is, motivated a softer alternative
- ref: kubernetes-sigs/prow#478
- relevance: By the same author (kaovilai). Repurposes `exclude` to actively remove protection. Has `lgtm`, all CI green. Maintainer initially moved to close it 2026-08-13T12:12:31Z (semantics change + no more branchprotector investment desired), but the subsequent exchange left him "softening" toward a separate additive field instead of merging #478 as-is. Also bypasses `equalBranchProtections` dedup — flagged as a concern independent of which approach proceeds.

### [related-issue] #846 — move away from branch protection towards rulesets
- ref: kubernetes-sigs/prow#846
- relevance: Cited by the maintainer as the reason not to invest further in branchprotector's feature set, and as the intended home for the config-surface lessons learned in this thread. No committed timeline as of this triage.

## Checked
- `Exclude` field documented as "branches that should be excluded from the protection policy" — ambiguous, docstring not yet updated
- `Unmanaged` confirmed to be a per-branch-name boolean, not a regex list — no existing pattern-based "don't manage" mechanism beyond `exclude` itself
- `Include` filtering has the same structural gap as `Exclude` (branches that stop matching keep existing protection) — not raised in the issue, not addressed by any proposed approach
- Existing tests only verify excluded branches don't receive protection, not that existing protection is removed
- Fake test client's `GetBranches` correctly simulates `Protected=true` for `onlyProtected=true` call
- Current `exclude` behavior has been stable for years and is, per the maintainer, intentional
- Line numbers for all cited code reconfirmed against current `main_sha` (`cced76d90`) — unchanged since original triage
- Full issue comment history re-read end to end; no linked/cross-referenced PRs beyond #478 and #846
- Maintainer's position evolved materially during the thread: from firm close (2026-08-13T12:12:31Z) to "softening" toward a new additive field (2026-08-13T13:18:08Z) after the bot-branch bootstrapping scenario was explained

## Next steps
- Do not merge PR #478 as-is — maintainer has ruled out changing `exclude`'s semantics
- Decide: commit to a new additive, pattern-based `protect: false` field (backwards compatible, solves the bot-branch scenario, reuses #478's removal machinery through a dedup-aware path) versus holding the status quo (close, redirect effort to rulesets #846)
- Regardless of that decision: land a docstring clarification on `Exclude` (`pkg/config/branch_protection.go:54-56`) stating it behaves as a pattern-scoped `unmanaged: true`, not a pattern-scoped `protect: false` — low-cost, unblocked either way
- Decide PR #478's fate explicitly once the above is resolved (close outright vs. leave open pending rework)
- Labels applied by maintainer so far: `area/branchprotector` (added), `lifecycle/rotten` (removed)

## Open questions
- Will the maintainer commit to the new additive field, or hold the close/redirect-to-rulesets position? This is the single blocking decision.
- If the additive field proceeds: what name avoids repeating the `exclude`/`unmanaged` confusion?
- Should `Include` receive the same retroactive-removal treatment for consistency, or is that explicitly out of scope?
- Is there a target timeline for the rulesets migration (#846) that would change the urgency calculus here?
- Should PR #478 be closed outright now that repurposing `exclude` is ruled out, or left open pending a possible rework?
