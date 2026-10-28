---
issue: kubernetes-sigs/prow#391
title: "assign: add configuration per repo so that only org users can assign issues"
state: open
labels: kind/feature, area/plugins, lifecycle/stale
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:31:54Z
verdict: accepted
---

## Findings

### [cause] Plugin was never built with a config surface
- detail: `helpProvider` explicitly states the plugin is unconfigurable, and `handle()` gates on nothing but GitHub's own assignee-eligibility rules (which restrict who can be *assigned*, not who can *do the assigning*).
- evidence: `pkg/plugins/assign/assign.go:41` — comment "The Config field is omitted because this plugin is not configurable."; `pkg/plugins/assign/assign.go:113-152` (`handle`) calls `h.add`/`h.remove` unconditionally for any commenter.

### [related-code] Config struct + per-org/repo getter pattern to follow
- where: `pkg/plugins/config.go:360-380` (`Approve` struct), `pkg/plugins/config.go:1029` (`ApproveFor`)
- excerpt: |
    func (c *Configuration) ApproveFor(org, repo string) *Approve {
    	fullName := fmt.Sprintf("%s/%s", org, repo)
    	a := func() *Approve {
    		// First search for repo config
    		for _, approve := range c.Approve {
    			if !sets.New[string](approve.Repos...).Has(fullName) {
    				continue
    			}
    			return &approve
    		}
    		// If you don't find anything, loop again looking for an org config
    		for _, approve := range c.Approve {
    			if !sets.New[string](approve.Repos...).Has(org) {
    				continue
    			}
    			return &approve
    		}

### [related-code] Existing per-repo OnlyOrgMembers analogue
- where: `pkg/plugins/config.go:554-582` (`Trigger` struct)
- excerpt: |
    // OnlyOrgMembers requires PRs and/or /ok-to-test comments to come from org members.
    // By default, trigger also include repo collaborators.
    OnlyOrgMembers bool `json:"only_org_members,omitempty"`

### [related-code] Reusable org-membership/trust check
- where: `pkg/plugins/trigger/trigger.go:245`
- excerpt: |
    func TrustedUser(ghc trustedUserClient, onlyOrgMembers bool, trustedApps []string, trustedOrg, user, org, repo string) (TrustedUserResponse, error) {
- relevance: implements exactly the "is this user an org member / collaborator / from a trusted app" check the assign gate would need; avoids reimplementing trust logic. Reusing it means importing `pkg/plugins/trigger` from `pkg/plugins/assign`, or extracting the logic to a neutral shared location — an open design question.

### [related-code] githubClient interface needs IsMember
- where: `pkg/plugins/assign/assign.go:63-70` (`githubClient` interface)
- excerpt: |
    type githubClient interface {
    	AssignIssue(owner, repo string, number int, logins []string) error
    	UnassignIssue(owner, repo string, number int, logins []string) error
    	RequestReview(org, repo string, number int, logins []string) error
    	UnrequestReview(org, repo string, number int, logins []string) error
    	CreateComment(owner, repo string, number int, comment string) error
    }
- relevance: needs an `IsMember(org, user string) (bool, error)` method added to call into the trust check; already implemented on the real GitHub client and used elsewhere (e.g. `trigger`).

### [related-code] GenericCommentEvent lacks issue labels (scopes the follow-on idea out)
- where: `pkg/github/types.go:1323-1341`
- relevance: the 2026-04-14 follow-up comment's idea (warn non-org-members self-assigning issues that lack a `good-first-issue` label) needs the issue's labels, which aren't on `GenericCommentEvent` — would require an extra GitHub API call. Confirms that idea is materially larger than the base ask and should not be bundled into it. Label constant already exists: `labels.GoodFirstIssue` (`pkg/labels/labels.go:34`).

### [related-issue] Referenced drive-by-assign example
- ref: kubevirt/kubevirt#14085 (comment https://github.com/kubevirt/kubevirt/issues/14085#issuecomment-2694071352)
- relevance: the concrete incident that prompted this issue — GSoC participants self-assigning issues not vetted as beginner-friendly.

## Checked

- No existing `Assign`-named field in `plugins.Configuration` (`pkg/plugins/config.go:52-100`) — confirms the feature doesn't already exist under another name.
- `pkg/plugins/assign/assign_test.go`'s `fakeClient` (line 28) has no `IsMember` mock yet — trust-check testing would need new scaffolding, mirrored on `trigger_test.go`'s approach.
- No open PR currently references or closes this issue (checked issue timeline/cross-references).
- The default-on vs. opt-in question raised by `petr-muller` in a 2025-03-13 comment was never resolved in-thread.
- Issue has cycled through `lifecycle/stale`/`rotten`/close/reopen multiple times (2025-06-18 through 2026-07-13), each time kept alive by `dhiller` or `petr-muller` manually — still legitimate and actionable, not abandoned.

## Next steps

- Apply `help-wanted` label (not `good-first-issue` — the trust-check-reuse-vs-duplicate decision and the unresolved default-on/opt-in question push it a notch above a pure first issue).
- Post a scoping comment: confirm the base ask (opt-in `only_org_members` gate reusing `trigger.TrustedUser`) is what's tracked here, and suggest splitting the 2026-04-14 good-first-issue-warning-comment idea into its own issue.
- Get maintainer/sig-contribex input on default-on vs. opt-in before a PR lands, so an accepted PR's scope doesn't get renegotiated mid-review.
- Remove `lifecycle/stale` (this triage constitutes fresh activity).

## Open questions

- Should `OnlyOrgMembers` for `/assign` default to on (behavior change for all repos, needs sig-contribex sign-off) or ship opt-in per repo first?
- Should the gate apply to self-assignment (`/assign` with no args) or only to assigning *others*? The original report is about assigning others; a separate commenter (`mohit-nagaraj`) also asks about blocking *re*-assignment by non-owners.
- Should the trust-check reuse `trigger.TrustedUser` directly (new import edge `assign` → `trigger`) or should the shared logic be extracted to a neutral location?
