---
issue: kubernetes-sigs/prow#780
title: "Smart blunderbuss reviewer selection using git blame data"
state: open
labels:
main_sha: ae8e2d87967f0a2b45cfed2c514f5ec91b964596
triaged_at: 2026-07-03T12:51:48Z
verdict: needs-discussion
refresh_log:
  - previous: 2026-06-30T15:50:17Z
    summary: "PR #787 submitted by smg247 implementing a separate `rifle` plugin. Author chose separate-plugin approach (option B). Still uses original scoring formula without addressing maintainer concerns."
---

## Findings

### [cause] Random reviewer selection ignores contributor expertise
- detail: Blunderbuss selects reviewers via `PopRandom()` from OWNERS-listed candidates. Within each priority layer, selection is purely random regardless of who actually wrote or maintains the changed code.
- evidence: `pkg/plugins/blunderbuss/blunderbuss.go:433-458`

### [related-code] findReviewer — the random selection point
- where: `pkg/plugins/blunderbuss/blunderbuss.go:433-458`
- excerpt: |
    func findReviewer(ghc githubClient, log *logrus.Entry, useStatusAvailability bool, busyReviewers *sets.Set[string], targetSet *layeredsets.String) string {
        if !useStatusAvailability {
            return targetSet.PopRandom()
        }
        for targetSet.Len() > 0 {
            candidate := targetSet.PopRandom()

### [related-code] getReviewers — multi-stage reviewer collection
- where: `pkg/plugins/blunderbuss/blunderbuss.go:380-428`
- excerpt: |
    func getReviewers(rc reviewersClient, ghc githubClient, log *logrus.Entry, author string, files []github.PullRequestChange, minReviewers int, useStatusAvailability bool) ([]string, []string, error) {
        // first build 'reviewers' by taking a unique reviewer from each OWNERS file.
        // ...
        // now ensure that we request review from at least minReviewers reviewers. Favor leaf reviewers.

### [related-code] PopRandom — the randomization primitive
- where: `pkg/layeredsets/string.go:99-111`
- excerpt: |
    func (s String) PopRandom() string {
        for _, layer := range s {
            if layer.Len() > 0 {
                list := sets.List(layer)
                sort.Strings(list)
                sel := list[rand.Intn(len(list))]
                s.Delete(sel)
                return sel
            }
        }
        return ""
    }

### [related-code] Blunderbuss configuration struct
- where: `pkg/plugins/config.go:196-223`
- excerpt: |
    type Blunderbuss struct {
        ReviewerCount         *int         `json:"request_count,omitempty"`
        MaxReviewerCount      int          `json:"max_request_count,omitempty"`
        ExcludeApprovers      bool         `json:"exclude_approvers,omitempty"`
        UseStatusAvailability bool         `json:"use_status_availability,omitempty"`
        IgnoreDrafts          bool         `json:"ignore_drafts,omitempty"`
        IgnoreAuthors         []string     `json:"ignore_authors,omitempty"`
        WaitForStatus         *ContextMatch `json:"wait_for_status,omitempty"`
    }

### [related-code] Blunderbuss test suite
- where: `pkg/plugins/blunderbuss/blunderbuss_test.go`
- excerpt: 1124 lines covering reviewer assignment from OWNERS, approver fallback, ExcludeApprovers, MaxReviewerCount, UseStatusAvailability, IgnoreDrafts, IgnoreAuthors, WaitForStatus, and required reviewers. Uses fakeGitHubClient and fakeOwnersClient.

### [related-pr] rifle plugin implementation
- ref: kubernetes-sigs/prow#787
- relevance: Author (smg247) chose the separate-plugin approach. PR adds a new `rifle` plugin with blame-based scoring, a `GetBlame` GitHub client method (GraphQL with Apps auth), shared reviewer logic extracted to `pkg/reviewer/reviewer.go`. Touches 17 files. Still uses original scoring formula `(lines * 10) + (recency * 5) + owner_bonus` which has not been revised per maintainer feedback.

### [related-issue] advisory_approvers OWNERS field
- ref: kubernetes-sigs/prow#784
- relevance: Related to OWNERS-based reviewer/approver semantics — changing how people are listed and selected from OWNERS files.

## Checked

- No existing blame API usage anywhere in the Prow Go codebase (only a URL in a comment in `cmd/gangway/main.go`)
- No other plugins implement smart/weighted reviewer selection
- GitHub GraphQL client (`shurcooL/githubv4`) is available and already used by blunderbuss for availability checks (`isUserBusy()`)
- `layeredsets.String` enforces layer-based priority (layer 0 first, then 1, then 2) — any scoring-based replacement must interact with or replace this layering

## Next steps

- Review PR #787 — the author chose approach B (separate plugin named `rifle`) and has a working implementation
- During review, check whether the scoring formula has been revised to address the reviewer-vs-approver weighting concern (original `owner_bonus` of +3 approver / +2 reviewer was flagged as backwards)
- Evaluate the shared `pkg/reviewer/reviewer.go` extraction — does it introduce coupling between blunderbuss and rifle?
- Assess GitHub API rate limit impact of per-file blame queries on large PRs
- Apply labels: `kind/feature`, `area/plugins`

## Open questions

- The scoring formula in PR #787 still uses the original `owner_bonus` values — has the maintainer concern been addressed?
- How does the `rifle` plugin handle PRs touching many files from an API rate limit perspective?
- Is the `pkg/reviewer/reviewer.go` shared package the right abstraction boundary?
- Should blunderbuss and rifle be declared mutually exclusive at the config validation level?
