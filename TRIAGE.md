---
issue: kubernetes-sigs/prow#438
title: "Add regex support for branch matching in Tide status controller"
state: open
labels: kind/feature, area/tide
main_sha: 61402617b3311e0ea6c0274b68ae71e9a524ab8b
triaged_at: 2026-10-20T14:35:20Z
verdict: accepted
refresh_log:
  - at: 2026-10-19T13:40:47Z
    since: 2026-06-02T09:01:39Z
    summary: "Issue remains open; stale lifecycle activity and four cross-referenced OpenShift release PRs were incorporated, including an unmerged regex-config attempt."
  - at: 2026-10-20T14:35:20Z
    since: 2026-10-19T13:40:47Z
    summary: "kaovilai removed the lifecycle/stale label; issue remains open with no new cross-references or main-branch changes."
---

## What the issue reports

The issue requests regular-expression matching for target branches in Tide status queries.

Since previous triage:
- The issue remains open. `lifecycle/stale` was added, and k8s-triage-robot posted the standard stale notice on 2026-10-19.
- Four OpenShift release PRs were cross-referenced: #83619 merged with branch-specific Tide config changes; #84020 merged as repository onboarding; #84118 closed unmerged after exact branch exclusions; #85777 closed unmerged after proposing `includedBranchesRe` for app-scoped branches.
- In #85777, a CodeRabbit review flagged `includedBranchesRe` as unsupported; upstream Prow main still has no corresponding Tide query fields.

Since previous triage (2026-10-19T13:40:47Z):
- On 2026-10-20 at 02:25:54Z, kaovilai commented `/remove-lifecycle stale`; the issue remains open and the stale label is gone.

## Findings

### [cause] GitHub Search API base: qualifier is exact-match only
- detail: Tide drives PR discovery via the GitHub Search API, translating `includedBranches`/`excludedBranches` entries into `base:"X"` / `-base:"X"` qualifiers. The GitHub Search API does not support regex for the `base:` qualifier, so patterns cannot be passed through directly.
- evidence: `pkg/config/tide.go:620-625`

### [cause] Branchprotector analogy is architecturally misleading
- detail: Branchprotector enumerates all branches via the GitHub Branches API and filters locally with compiled `*regexp.Regexp`. Tide has no equivalent enumeration step; it relies on GitHub to do the filtering server-side. The two components are not analogous.
- evidence: `cmd/branchprotector/protect.go:312-346`

### [related-code] TideQuery branch fields
- where: `pkg/config/tide.go:546-562`
- excerpt: |
    ExcludedBranches []string `json:"excludedBranches,omitempty"`
    IncludedBranches []string `json:"includedBranches,omitempty"`

### [related-code] constructQuery branch qualifier generation
- where: `pkg/config/tide.go:620-625`
- excerpt: |
    for _, b := range tq.ExcludedBranches {
        queryString = append(queryString, fmt.Sprintf("-base:\"%s\"", b))
    }
    for _, b := range tq.IncludedBranches {
        queryString = append(queryString, fmt.Sprintf("base:\"%s\"", b))
    }

### [related-code] requirementDiff branch scoring (status only, not merge decisions)
- where: `pkg/tide/status.go:149-155`
- excerpt: |
    targetBranchDenied := slices.Contains(q.ExcludedBranches, string(pr.BaseRef.Name))
    targetBranchAllowed := len(q.IncludedBranches) == 0
    if slices.Contains(q.IncludedBranches, string(pr.BaseRef.Name)) {
        targetBranchAllowed = true
    }

### [related-code] branchprotector regex branch filtering
- where: `cmd/branchprotector/protect.go:312-346`
- excerpt: |
    var branchInclusions *regexp.Regexp
    if len(repo.Policy.Include) > 0 {
        branchInclusions, err = regexp.Compile(strings.Join(repo.Policy.Include, `|`))
    }
    // then filters against GetBranches() results locally

### [related-issue] GitHub Search API improvements
- ref: kubernetes-sigs/prow#482
- relevance: Explores GitHub's improved Search API; if GitHub adds regex support to base: qualifier the implementation becomes trivial.

### [related-pr] OpenShift config attempt to use regex branch matching
- ref: openshift/release#85777
- relevance: Proposed `includedBranchesRe` for app-scoped branch names, but closed unmerged on 2026-10-08; a CodeRabbit review flagged the selector as unsupported. This is a configuration attempt, not an implementation in Prow.

## Checked
- GitHub Search API `base:` qualifier is exact-match only — confirmed by reading `constructQuery()` and the GitHub Search API docs reference in the source comments (`pkg/config/tide.go:545`)
- `requirementDiff()` branch matching drives status display only, not merge decisions — a fix there alone would be insufficient
- No existing PRs are working on this feature
- branchprotector regex works via Branches API enumeration + local filtering — not applicable to Tide's search-based architecture
- Upstream main advanced to `61402617b3311e0ea6c0274b68ae71e9a524ab8b`; its Tide query code still has no regex branch fields, and the intervening changes in `pkg/config/tide.go` and `pkg/tide/status.go` do not implement them

## Next steps
- Apply `help-wanted` label
- Post a comment summarizing the GitHub Search API constraint and the three implementation approaches (enumerate-and-expand, local post-filter, new config fields) so contributors understand the design decision needed before coding
- Monitor issue #482 for GitHub Search API improvements that could simplify the implementation
- Require a design discussion comment or GH Discussion before accepting implementation PRs

## Open questions
- Which implementation approach do Tide maintainers prefer: enumerate-and-expand (correct semantics, extra Branches API calls per sync), local post-filter (no extra calls, larger PR result sets), or separate new config fields?
- Is the additional GitHub Branches API call overhead acceptable given Tide's existing rate limit budget on large deployments?
- Should new regex fields be named `includedBranchPatterns`/`excludedBranchPatterns` or something else?
