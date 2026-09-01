---
issue: kubernetes-sigs/prow#910
title: "mistakenly failed to remove `approved` after requirements changed"
state: open
labels:
main_sha: f48e269a9dd8ab5282b86fc3df28187d8b972981
triaged_at: 2026-09-01T12:37:34Z
verdict: accepted
---

## Findings

### [reproducibility] External PR event confirms stale bot identity classification
- detail: The only `approved` label-add event on `kubernetes/enhancements#5923` was performed by `k8s-ci-robot` on 2026-02-16. The affected Prow instance now authenticates as a GitHub App, so its current bot identity differs from the former PAT account.
- evidence: `gh api repos/kubernetes/enhancements/issues/5923/events`; kubernetes-sigs/prow#910#issuecomment-5484362158.

### [cause] Current-identity-only check treats a former Prow account as human
- detail: `WasLabelAddedByHuman` examines the last `approved` label event and asks only whether its actor equals the currently authenticated bot login. `BotUserCheckerWithContext` compares the candidate to that one current identity (stripping `[bot]` only when app-auth is enabled). Thus the historic `k8s-ci-robot` event is classified as human after the PAT-to-App migration.
- evidence: `pkg/github/client.go:1310-1325`, `pkg/github/client.go:3157-3176`.

### [cause] Classification invokes an unconditional approval bypass
- detail: The approve plugin assigns that classification to `ManuallyApproved`; `IsApproved` returns `RequirementsMet() || ManuallyApproved()`. Once stale bot activity is misclassified as manual, the plugin preserves `approved` even after the changed file set no longer satisfies OWNERS requirements.
- evidence: `pkg/plugins/approve/approve.go:439`, `pkg/plugins/approve/approvers/owners.go:556-563`.

### [related-code] Label reconciliation is otherwise correct
- where: `pkg/plugins/approve/approve.go:490-505`
- excerpt: |
    if !approversHandler.IsApproved() {
        if hasApprovedLabel {
            if err := ghc.RemoveLabel(pr.org, pr.repo, pr.number, labels.Approved); err != nil {
                ...
            }
        }
    } else if !hasApprovedLabel {
        if err := ghc.AddLabel(pr.org, pr.repo, pr.number, labels.Approved); err != nil {
            ...
        }
    }

### [related-code] Manual bypass has no configuration switch
- where: `pkg/plugins/approve/approve.go:439`, `pkg/plugins/approve/approvers/owners.go:561-563`
- excerpt: |
    approversHandler.ManuallyApproved = humanAddedApproved(ghc, log, pr.org, pr.repo, pr.number, hasApprovedLabel)

    // If a human manually added the approved label, this returns true, ignoring normal approval rules.
    func (ap Approvers) IsApproved() bool {
        return ap.RequirementsMet() || ap.ManuallyApproved()
    }

### [related-issue] Maintainer diagnosis and intended remediation
- ref: kubernetes-sigs/prow#910
- relevance: BenTheElder confirmed the PAT-to-GitHub-App transition, removed the affected label manually, and proposed making manual-label bypass optional and disabling it in Kubernetes.

## Checked
- Retrieved the affected PR's issue-event history; its `approved` label was added by `k8s-ci-robot`, not the PR author.
- Read the current approve reconciliation, `ManuallyApproved`, and GitHub bot-identity code.
- Searched approve-plugin configuration and found no existing manual-approval bypass option.
- Confirmed `kind/bug` and `area/plugins` exist; the issue currently has no labels.

## Next steps
- Accept as a bug; add `kind/bug` and `area/plugins`.
- Add an approve-plugin configuration option governing whether a manually applied `approved` label bypasses normal approval requirements; preserve existing behavior only where explicitly enabled.
- Disable that option in Kubernetes' Prow configuration.
- Add tests covering a historic bot actor that is no longer the configured/current bot identity, and both enabled and disabled bypass behavior.
- Consider a migration-safe bot-identity allowlist separately if installations require retaining historic bot-generated labels without treating them as manual.

## Open questions
- Should the new option default to the present permissive behavior for backwards compatibility, or to disabled for safer approval semantics?
- Is an explicit list of former bot identities desirable, or should the Kubernetes configuration change be the complete remediation?
