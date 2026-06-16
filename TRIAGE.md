---
issue: kubernetes-sigs/prow#279
title: "alert prow plugin for no cla PR merges"
state: open
labels: kind/feature, help wanted, lifecycle/frozen, area/plugins
main_sha: 71428b9c282ee8c9e7e9512068fccce86e7915da
triaged_at: 2026-06-15T16:38:01Z
verdict: accepted
refresh_log:
  - prev: 2026-06-01T14:19:36Z
    summary: "petr-muller replied to AaruniAggarwal confirming contribution and choosing to extend slackevents; area/plugins label added"
---

## Findings

### [cause] No PullRequestHandler in slackevents to detect PR merges and check CLA status
- detail: The `slackevents` plugin registers only PushEvent and GenericComment handlers. It has no mechanism to react to PR merge events and inspect commit statuses. A new `PullRequestHandler` is needed to detect merged PRs and check the `"EasyCLA"` status context.
- evidence: `pkg/plugins/slackevents/slackevents.go:54-57`

### [related-code] slackevents plugin — existing Slack alert infrastructure
- where: `pkg/plugins/slackevents/slackevents.go`
- excerpt: |
    func init() {
        plugins.RegisterPushEventHandler(pluginName, handlePush, helpProvider)
        plugins.RegisterGenericCommentHandler(pluginName, handleComment, helpProvider)
    }

### [related-code] CLA plugin — defines "EasyCLA" status context
- where: `pkg/plugins/cla/cla.go:34-37`
- excerpt: |
    const (
        pluginName     = "cla"
        claContextName = "EasyCLA"
        maxRetries     = 5
    )

### [related-code] CLA labels
- where: `pkg/labels/labels.go:29-30`
- excerpt: |
    ClaNo  = "cncf-cla: no"
    ClaYes = "cncf-cla: yes"

### [related-code] GetCombinedStatus — GitHub API for querying commit statuses
- where: `pkg/github/client.go` (interface at lines 152-165, impl at lines 2908-2930)
- excerpt: |
    GetCombinedStatus(org, repo, ref string) (*CombinedStatus, error)

### [related-code] branchcleaner — canonical pattern for detecting merged PRs
- where: `pkg/plugins/branchcleaner/branchcleaner.go:82-85`
- excerpt: |
    if pre.Action != github.PullRequestActionClosed || !pre.PullRequest.Merged {
        return nil
    }

### [related-code] RegisterPullRequestHandler — plugin registration point
- where: `pkg/plugins/plugins.go:128-135`
- excerpt: |
    type PullRequestHandler func(Agent, github.PullRequestEvent) error
    func RegisterPullRequestHandler(name string, fn PullRequestHandler, help HelpProvider) {
        pluginHelp[name] = help
        pullRequestHandlers[name] = fn
    }

### [related-code] Slack config struct and MergeWarning
- where: `pkg/plugins/config.go:597-600` (Slack), `pkg/plugins/config.go:809-818` (MergeWarning)
- excerpt: |
    type Slack struct {
        MentionChannels []string       `json:"mentionchannels,omitempty"`
        MergeWarnings   []MergeWarning `json:"mergewarnings,omitempty"`
    }

### [related-code] HasLabel helper for label checking
- where: `pkg/github/helpers.go:35-42`
- excerpt: |
    func HasLabel(label string, issueLabels []Label) bool {
        for _, l := range issueLabels {
            if strings.EqualFold(l.Name, label) { return true }
        }
        return false
    }

### [related-issue] kubernetes/community#8447 — relaxed help-wanted guidelines
- ref: kubernetes/community#8447
- relevance: Merged 2025-05-14. Cross-referenced by BenTheElder when adding `help wanted` label to this issue.

## Checked
- No existing plugin handles CLA-merge alerts
- `slackevents` only registers PushEvent and GenericComment handlers (no PullRequestHandler)
- `GetCombinedStatus` API is available and already used by the CLA plugin
- `"EasyCLA"` status context is the authoritative CLA signal (per BenTheElder and cla plugin code)
- `PullRequestEvent` payload includes `Head.SHA` regardless of merge strategy (merge, squash, or rebase)
- Prometheus metrics approach was discussed and rejected due to high cardinality concerns (BenTheElder, 2024-09-23)

## Next steps
- ~~Reply to AaruniAggarwal confirming the issue is open for contribution~~ Done (petr-muller, 2026-06-15)
- ~~Decide and communicate: extend `slackevents` or create a standalone plugin~~ Decided: extend `slackevents` (petr-muller, 2026-06-15)
- ~~Apply `area/plugins` label~~ Done (petr-muller, 2026-06-15)
- Define v1 scope: check `Head.SHA` status from `PullRequestEvent`, document squash/rebase limitation, defer handling to follow-up
- Await PR from AaruniAggarwal

## Open questions
- ~~Extend `slackevents` vs. new standalone plugin?~~ Resolved: extend `slackevents` (petr-muller, 2026-06-15)
- Should v1 check commit status only, label only, or both? BenTheElder says status is source of truth; label could be secondary signal
- What should the Slack message format look like? Contributor can propose in PR
