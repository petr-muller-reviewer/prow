---
issue: kubernetes-sigs/prow#391
title: "assign: add configuration per repo so that only org users can assign issues"
state: open
labels: kind/feature, area/plugins, lifecycle/rotten
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-10-28T00:38:36Z
verdict: accepted
refresh_log:
  - at: 2026-10-28T00:38:36Z
    since: 2026-07-27T00:31:54Z
    summary: "Recorded the Oct 17 lifecycle rotation and reviewed open PR #755's proposed implementation."
---

## What the issue reports

The issue asks for a repository-level setting that limits `/assign` on issues to members of the owning organization.

Since previous triage:
- On 2026-10-17, k8s-triage-robot posted the lifecycle stale-cycle notice; `lifecycle/stale` was removed and `lifecycle/rotten` added. The issue remains open.
- PR #755 remains open (created 2026-06-14 and updated 2026-09-30). Its description claims `Fixes: #391`; the proposal adds per-org/repo `block` or `warn` behavior with optional exempt labels, checks the commenter's org membership for issue assignments including self-assignment, and leaves pull requests and `/unassign` unrestricted. No new cross-reference event appeared on the issue timeline since the prior triage.

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

### [related-pr] Open implementation for issue #391
- ref: kubernetes-sigs/prow#755
- relevance: Open PR, last updated 2026-09-30, proposes per-org/repo restriction configuration with `block`/`warn` actions and label exemptions. It checks the commenter's membership directly with `IsMember`, including for self-assignment; pull requests and `/unassign` are excluded. This matches the requested configuration and membership gate, while choosing direct membership checking over `trigger.TrustedUser` and adding the action, exemption, and self-assignment decisions.

### [related-issue] Referenced drive-by-assign example
- ref: kubevirt/kubevirt#14085 (comment https://github.com/kubevirt/kubevirt/issues/14085#issuecomment-2694071352)
- relevance: the concrete incident that prompted this issue — GSoC participants self-assigning issues not vetted as beginner-friendly.

## Checked

- No existing `Assign`-named field in `plugins.Configuration` (`pkg/plugins/config.go:52-100`) — confirms the feature doesn't already exist under another name.
- `pkg/plugins/assign/assign_test.go`'s `fakeClient` (line 28) has no `IsMember` mock yet — trust-check testing would need new scaffolding, mirrored on `trigger_test.go`'s approach.
- PR #755 is open and its description claims to fix #391; it was created before the prior triage and last updated 2026-09-30. The issue timeline contains no new cross-reference events since the prior triage.
- The default-on vs. opt-in question raised by `petr-muller` in a 2025-03-13 comment was never resolved in-thread.
- Issue has cycled through `lifecycle/stale`/`rotten`/close/reopen multiple times (2025-06-18 through 2026-07-13), each time kept alive by `dhiller` or `petr-muller` manually — still legitimate and actionable, not abandoned.

## Next steps

- Apply `help-wanted` label (not `good-first-issue` — the trust-check design and remaining scope decisions make this more than a pure first issue).
- Review/track PR #755. Its per-repo config and org-membership gate match the core request; settle whether self-assignment, `warn` versus `block`, and exempt-label behavior belong in this issue before merging.
- Confirm the opt-in policy and whether direct `IsMember` is sufficient or the broader `trigger.TrustedUser` behavior is required.
- Compare PR #755's warning and exempt-label behavior with the 2026-04-14 good-first-issue-warning idea; decide whether that scope stays here or should be split.
- Remove `lifecycle/rotten` while maintainers are actively considering the open PR.

## Open questions

- Should the restriction ship opt-in per repo, as PR #755 proposes, or default on (which changes behavior across repos and needs sig-contribex sign-off)?
- Should the gate apply to self-assignment as well as assigning others? PR #755 applies it to both; the original report focuses on assigning others, and `mohit-nagaraj` also asked about blocking reassignment by non-owners.
- Is direct `IsMember` sufficient, as PR #755 implements, or should the check reuse `trigger.TrustedUser` or shared trust logic?
- Should this issue include PR #755's `warn`/`block` choices and exempt-label behavior, including the good-first-issue warning idea?
