---
issue: kubernetes-sigs/prow#1011
title: "cherrypicker: support applying PRs as a single squashed diff instead of per-commit git am"
state: open
labels: sig/testing
main_sha: 9851264902811315625eb653081f981dfb499289
triaged_at: 2026-10-15T23:34:25Z
verdict: accepted
advice:
  advised_at: 2026-10-15T23:36:42Z
  based_on_triaged_at: 2026-10-15T23:34:25Z
---

# Triage

## Verdict

**accepted** — This is an actionable feature request for Prow's cherrypicker external plugin. The per-commit `git am` behavior is confirmed in the current code, and the requested aggregate-diff mode could reduce conflict work. It needs repo-scoped configuration and decisions about source blobs and commit metadata before implementation.

## What the issue reports

- The cherrypicker currently fetches GitHub's patch and replays each source commit with `git am --3way`.
- A conflict in an intermediate commit can stop the cherry-pick even when resolving the PR's net change once would be practical.
- The author requests an optional squashed-diff mode that creates a single commit and preserves original authorship with co-author trailers.
- The issue cites kubernetes-sigs/kueue#17193 and PR #17200 as prior art for a manual cherry-pick script.

## Findings

### [cause] The handler replays patch emails one commit at a time
- detail: The handler passes the fetched patch to `r.Am`, which runs `git am --3way`; a failure aborts the current apply and the handler reports a conflict. This is the current design, not an unintended regression.
- evidence: `cmd/external-plugins/cherrypicker/server.go:615-631`; `pkg/git/v2/interactor.go:425-438`.

### [related-code] Cherrypicker apply path
- where: `cmd/external-plugins/cherrypicker/server.go:569-616`
- excerpt: |
    // Apply the patch.
    if err := r.Am(localPath); err != nil {
        errs := []error{fmt.Errorf("failed to `git am`: %w", err)}

### [related-code] GitHub client already provides a pull request diff endpoint
- where: `pkg/github/client.go:2559-2590`
- excerpt: |
    accept:    "application/vnd.github.diff",
    method:    http.MethodGet,
    path:      fmt.Sprintf("/repos/%s/%s/pulls/%d", org, repo, number),
- detail: The cherrypicker's local `githubClient` interface currently exposes the patch method, not the diff method.

### [related-code] Existing external-plugin configuration is routing-only
- where: `pkg/plugins/config.go:171-182`
- excerpt: |
    type ExternalPlugin struct {
        Name     string   `json:"name"`
        Endpoint string   `json:"endpoint,omitempty"`
        Events   []string `json:"events,omitempty"`
    }
- detail: The cherrypicker's CLI flags are process-wide. A single boolean would affect every repository handled by that server. The issue needs a repo-scoped setting; the existing `external_plugins` entries do not pass arbitrary per-repository options to the server.

### [related-code] Git publisher can commit but has no sign-off option
- where: `pkg/git/v2/publisher.go:52-68`
- excerpt: |
    {"add", "--all"},
    {"commit", "--message", title, "--message", body, "--author", fmt.Sprintf("%s <%s>", name, email)},
- detail: A one-commit mode needs an explicit decision about commit author, co-author trailers, and DCO sign-off, plus any publisher API support required.

### [related-issue] Existing Prow cherrypicker issues are adjacent, not duplicates
- ref: kubernetes-sigs/prow#500, kubernetes-sigs/prow#745
- relevance: #500 concerns original commit IDs in commit messages; #745 concerns a CRLF-related false conflict. Neither requests applying the PR net diff as one commit.

### [related-issue] Kueue prior-art issue
- ref: kubernetes-sigs/kueue#17193
- relevance: Cross-referenced by this issue; it asks for a manual cherry-pick script to operate on a squashed commit.

### [related-pr] Kueue manual cherry-pick implementation
- ref: kubernetes-sigs/kueue#17200
- relevance: Open PR that applies each PR's net diff once in a manual script. It is useful prior art but does not implement Prow's external plugin behavior.

## Resolved

No open questions have been resolved through implementation or reporter follow-up. The maintainer clarified that the mode must be selectable per repository, not as one process-wide switch.

## Checked

- Read issue #1011, its comments, labels, and timeline. The issue is open, labeled `sig/testing`, and has an `/assign` comment; no Prow PR referencing #1011 surfaced in the search.
- Queried cross-references and searched Prow issues for `cherrypicker squash`, `cherrypicker git am`, and `cherrypicker diff`.
- Reviewed Prow issues #500, #745, and #544. #500 and #745 are related but distinct; #544 is a separate completed request-processing bug.
- Read the cherrypicker handler, its CLI options, the GitHub diff/patch methods, the Git interactor and publisher, relevant tests, plugin configuration, and component documentation.
- Existing tests cover the format-patch/`git am` path; no test currently covers aggregate-diff apply, one-commit output, or its authorship/sign-off semantics.
- Read Kueue issue #17193 and open PR #17200. Git documents that `git apply --3way` requires the patch's base blobs to be locally available ([git-apply documentation](https://git-scm.com/docs/git-apply)).

## Next steps

- Decide where repo-scoped opt-in configuration lives. Current `external_plugins` configuration only routes events; adding a setting requires a dedicated cherrypicker config/allowlist or additional config plumbing.
- Define how the handler will make the diff's base blobs available for three-way apply, including for merged PRs and PRs whose base branch has advanced.
- Agree on the single commit's message, author, co-author trailers, and sign-off behavior.
- Add coverage for single- and multi-commit PRs, conflicts and cleanup, per-repo mode selection, and unchanged behavior for unconfigured repos; update the cherrypicker docs.
- Consider applying `kind/feature` and `area/plugins`; retain the current `sig/testing` label.

## Advice

### Add the feature and area labels
`kind/feature` and `area/plugins` exist in the repository and are not on the issue. Keep the existing `sig/testing` label.

```sh
gh issue edit 1011 --repo kubernetes-sigs/prow --add-label "kind/feature,area/plugins"
```

No additional comment or milestone is suggested now. The issue remains open and assigned to `yindia`; the existing next steps capture the design questions. No Prow PR is linked or in flight. Kueue PR #17200 remains open as external prior art.

## Open questions

- Could we agree on the source and format for the per-repository opt-in? The existing external-plugin routing config does not provide custom settings to this server.
- Should the resulting commit include the PR author, each source commit author as `Co-authored-by`, and a `Signed-off-by` trailer?
- How should the diff's base blobs be fetched or retained so `git apply --3way` can use them on the target branch?
