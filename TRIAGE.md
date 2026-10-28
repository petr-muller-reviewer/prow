---
issue: kubernetes-sigs/prow#693
title: "Integrate with Netlify for /retest Prow command"
state: open
labels: kind/feature, lifecycle/stale, area/plugins
main_sha: 53a81071b1f5a432c33362a42aa7bc9837f7ed14
triaged_at: 2026-10-28T00:33:22Z
verdict: needs-discussion
refresh_log:
  - at: 2026-10-28T00:33:22Z
    since: 2026-05-30T13:22:53Z
    summary: "Recorded the stale label, confirmed PR #708 remains open, and added the related Kueue queued-deploy diagnostic."
advice:
  advised_at: 2026-10-05T23:34:56Z
  based_on_triaged_at: 2026-05-30T13:22:53Z
---

## What the issue reports

The request is for an authorized Prow user to retry a failed or stuck Netlify deploy preview for pull requests in `kubernetes/website` or `kubernetes/contributor-site` using `/retest`.

Since previous triage:
- The issue remains open. `lifecycle/stale` was applied by `k8s-triage-robot` on 2026-10-19 at 12:41:17Z; there have been no new human comments or cross-references on issue #693 since 2026-05-30.
- PR #708 remains open and was updated 2026-10-16. Its current implementation includes paginated deploy lookup, retries an errored preview with `/retest`, and offers `/netlify-rebuild` for an explicit retry. Its July discussion considered Kueue as another consumer and deferred Kueue-specific credential and configuration questions.
- The PR discussion cross-referenced Kueue #15975 on 2026-09-22. That report found an `enqueued` preview could look missing while GitHub had no deploy status; Kueue #17049 merged on 2026-10-20 to report the Netlify deploy state after the bounded wait. This is adjacent diagnostic work, not a change to #693's retry request.

## Findings

### [reproducibility] Feature gap, not a bug
- detail: `/retest` only re-triggers ProwJobs and (optionally) GitHub Actions. No mechanism to trigger other external CI systems such as Netlify.
- evidence: `pkg/plugins/trigger/generic-comment.go:38-196` — full retest flow; Netlify absent.

### [cause] Netlify API does not expose deploy preview rebuild endpoint
- detail: Deploy previews are triggered exclusively by GitHub push webhook events. No documented Netlify API endpoint exists to trigger a deploy preview rebuild for a specific PR. Build hooks (`POST /build_hooks/{id}`) and `POST /sites/{id}/builds` only produce branch deploys — a distinct concept that does not create the `deploy-preview-N` URL and does not update GitHub PR status checks.
- evidence: Netlify OpenAPI spec v2.53.0 (`open-api.netlify.com`) — no retry/rebuild-preview endpoint. Netlify docs for deploy types confirm deploy previews require a push event.

### [cause] Undocumented retry endpoint claimed by Caesarsage — unverified
- detail: On 2026-04-30 Caesarsage posted a spike result claiming `POST /api/v1/deploys/{deploy_id}/retry` reruns the same deploy preview for forked PRs and preserves PR preview identity. This endpoint does not appear in the official Netlify OpenAPI spec. Evidence is a single Netlify support thread from Nov 2023 where even the poster could not find it in official docs. If validated, feasibility shifts from Level 4 to Level 2-3.
- evidence: https://answers.netlify.com/t/how-to-retry-preview-build-after-branch-environment-variables-have-been-set/107149

### [cause] Netlify PAT has no granular scoping
- detail: Netlify Personal Access Tokens grant full account access equivalent to the creating user. There is no scope restriction to build-only or single-site. Build hook URLs (URL-as-secret) are safer but cannot trigger deploy previews. This is the security concern BenTheElder raised in the original test-infra issue.
- evidence: Netlify open-api issue #168 (open since ~2019), community threads on restricted tokens.

### [related-code] TriggerGitHubWorkflows — existing external CI pattern
- where: `pkg/plugins/trigger/generic-comment.go:168-196`
- excerpt: |
    if trigger.TriggerGitHubWorkflows && (pjutil.RetestRe.MatchString(textToCheck) || pjutil.TestAllRe.MatchString(textToCheck)) {
        failedRuns, err := c.GitHubClient.GetFailedActionRunsByHeadBranch(org, repo, pr.Head.Ref, headSHA)
        for _, run := range failedRuns {
            go func() { c.GitHubClient.TriggerFailedGitHubWorkflow(org, repo, runID) }()
        }
    }

### [related-code] Trigger plugin config struct
- where: `pkg/plugins/config.go:485-510`
- excerpt: |
    type Trigger struct {
        Repos []string
        TrustedApps []string
        TrustedOrg string
        IgnoreOkToTest bool
        TriggerGitHubWorkflows bool
    }

### [related-code] /retest regex and filter
- where: `pkg/pjutil/filter.go:33-36`
- excerpt: |
    var RetestRe = regexp.MustCompile(`(?m)^/retest\s*$`)
    var RetestRequiredRe = regexp.MustCompile(`(?m)^/retest-required\s*$`)

### [related-issue] Original feature request
- ref: kubernetes/test-infra#35103
- relevance: Source issue; contains BenTheElder's security analysis and lmktfy's original request. Went stale/rotten multiple times since Jul 2025. Caesarsage duplicated it here because GitHub doesn't allow cross-org issue transfers.

### [related-issue] Kueue: queued preview appears missing
- ref: kubernetes-sigs/kueue#15975
- relevance: A September 2026 report found a deploy could remain `enqueued` while GitHub showed no deploy status; Kueue #17049 (merged 2026-10-20) now reports Netlify's deploy state after the bounded wait. This is adjacent diagnostic work and does not retry a deploy or validate the retry endpoint proposed by Prow PR #708.

## Checked
- Netlify OpenAPI spec v2.53.0 — no retry/rebuild-preview endpoint documented
- `pkg/plugins/trigger/generic-comment.go` — full /retest flow; TriggerGitHubWorkflows is the existing pattern
- `pkg/plugins/config.go:139-149` — ExternalPlugin struct (alternative integration point)
- `pkg/plugins/plugins.go:176-213` — plugin registration and Agent struct
- `pkg/pjutil/filter.go`, `pkg/pjutil/pjutil.go` — retest regex, ProwJob creation
- Netlify community threads on build hooks, deploy previews, PAT scoping
- Netlify docs: build hooks, deploy types, manage deploys
- GitHub issue #693 state, labels, comments, and timeline; PR #708 current status, files, discussion, and diff (2026-10-28)
- kubernetes-sigs/kueue#15975 and merged #17049 for the separate queued-deploy diagnostic

## Next steps
- Validate whether `POST /api/v1/deploys/{deploy_id}/retry` actually preserves deploy preview identity and updates GitHub PR status. Caesarsage should provide a test result or curl transcript.
- If endpoint works: assess security model — is a full-access PAT acceptable, or should we wait for Netlify to add scoped tokens?
- If endpoint works and security is acceptable: implement following the TriggerGitHubWorkflows pattern in `generic-comment.go:168-196`, adding a `TriggerNetlifyBuildHooks` (or `TriggerNetlifyDeployPreviews`) config option to the Trigger struct.
- If endpoint is unreliable/undocumented: document workarounds (empty commit, webhook re-delivery) and leave issue open pending Netlify API improvements.

## Advice

Issue #693 remains open, assigned to Caesarsage, with the same `kind/feature` and `area/plugins` labels and no new issue comments since the saved triage. PR #708 is still open and directly implements this request. In PR discussion, Caesarsage reported live Netlify/GitHub API testing on 2026-05-07. On 2026-07-27, apullo777 and mimowo raised Kueue-specific concerns; Caesarsage said they would keep `/retest` quiet, use `/netlify-rebuild` for explicit rebuilds, handle deploy pagination, and defer Kueue-specific support until the generic behavior is reviewed. The current PR diff includes pagination and both commands. Review the existing PR rather than starting a duplicate implementation; settle the retry endpoint/token questions and the boundary between generic behavior and Kueue follow-up before merging.

### Advance review on PR #708

- Rationale: the PR is the implementation in flight, has end-to-end testing reported by its author, and has not merged. `pkg/plugins/OWNERS` names cjwagner and the ContribEx leads as required reviewers; the latest Prow approval notice requests cblecker as approver. The open discussion leaves endpoint/token acceptability and Kueue-specific scope for maintainers to decide.

```bash
gh pr comment 708 --repo kubernetes-sigs/prow --body "$(cat <<'EOF'
> *This was generated by AI during triage.*

Could @cjwagner, @MadhavJivrajani, @Priyankasaggu11929, and @cblecker coordinate the next review of #708? The PR now includes deploy pagination and keeps `/retest` quiet while providing `/netlify-rebuild`; Caesarsage also reported end-to-end testing against the real Netlify and GitHub APIs. Please confirm whether the retry endpoint and token model are acceptable, and whether Kueue support, canceled/skipped preview states, and per-repository credentials belong in this PR or should be follow-up work. Keep #693 open until the PR is reviewed, merged, and its behavior verified.
EOF
)"
```

## Open questions
- Does `POST /api/v1/deploys/{deploy_id}/retry` actually update GitHub PR status checks and preserve the deploy-preview-N URL? Can Caesarsage share a test transcript?
- How would Prow look up the `deploy_id` for a PR's latest deploy preview? Likely `GET /sites/{site_id}/deploys?branch=deploy-preview-{pr_number}` — needs verification.
- Is relying on an undocumented Netlify endpoint acceptable for a feature in an upstream project? What happens when Netlify changes it without notice?
- Should Netlify credentials (PAT or build hook URL) be stored in a Kubernetes Secret and referenced from the trigger plugin config? What's the precedent for other external credentials in Prow?
