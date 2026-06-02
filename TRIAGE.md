---
issue: kubernetes-sigs/prow#438
title: "Add regex support for branch matching in Tide status controller"
state: open
labels: kind/feature, area/tide
main_sha: 24f6b2904e9231919e1c8bb4b9258c648c82991d
triaged_at: 2026-06-02T09:01:39Z
verdict: accepted
---

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

## Checked
- GitHub Search API `base:` qualifier is exact-match only — confirmed by reading `constructQuery()` and the GitHub Search API docs reference in the source comments (`pkg/config/tide.go:545`)
- `requirementDiff()` branch matching drives status display only, not merge decisions — a fix there alone would be insufficient
- No existing PRs are working on this feature
- branchprotector regex works via Branches API enumeration + local filtering — not applicable to Tide's search-based architecture

## Next steps
- Apply `help-wanted` label
- Post a comment summarizing the GitHub Search API constraint and the three implementation approaches (enumerate-and-expand, local post-filter, new config fields) so contributors understand the design decision needed before coding
- Monitor issue #482 for GitHub Search API improvements that could simplify the implementation
- Require a design discussion comment or GH Discussion before accepting implementation PRs

## Open questions
- Which implementation approach do Tide maintainers prefer: enumerate-and-expand (correct semantics, extra Branches API calls per sync), local post-filter (no extra calls, larger PR result sets), or separate new config fields?
- Is the additional GitHub Branches API call overhead acceptable given Tide's existing rate limit budget on large deployments?
- Should new regex fields be named `includedBranchPatterns`/`excludedBranchPatterns` or something else?
