---
issue: kubernetes-sigs/prow#693
title: "Integrate with Netlify for /retest Prow command"
state: open
labels: kind/feature, area/plugins
main_sha: 53a81071b1f5a432c33362a42aa7bc9837f7ed14
triaged_at: 2026-05-30T13:22:53Z
verdict: needs-discussion
---

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

## Checked
- Netlify OpenAPI spec v2.53.0 — no retry/rebuild-preview endpoint documented
- `pkg/plugins/trigger/generic-comment.go` — full /retest flow; TriggerGitHubWorkflows is the existing pattern
- `pkg/plugins/config.go:139-149` — ExternalPlugin struct (alternative integration point)
- `pkg/plugins/plugins.go:176-213` — plugin registration and Agent struct
- `pkg/pjutil/filter.go`, `pkg/pjutil/pjutil.go` — retest regex, ProwJob creation
- Netlify community threads on build hooks, deploy previews, PAT scoping
- Netlify docs: build hooks, deploy types, manage deploys

## Next steps
- Validate whether `POST /api/v1/deploys/{deploy_id}/retry` actually preserves deploy preview identity and updates GitHub PR status. Caesarsage should provide a test result or curl transcript.
- If endpoint works: assess security model — is a full-access PAT acceptable, or should we wait for Netlify to add scoped tokens?
- If endpoint works and security is acceptable: implement following the TriggerGitHubWorkflows pattern in `generic-comment.go:168-196`, adding a `TriggerNetlifyBuildHooks` (or `TriggerNetlifyDeployPreviews`) config option to the Trigger struct.
- If endpoint is unreliable/undocumented: document workarounds (empty commit, webhook re-delivery) and leave issue open pending Netlify API improvements.

## Open questions
- Does `POST /api/v1/deploys/{deploy_id}/retry` actually update GitHub PR status checks and preserve the deploy-preview-N URL? Can Caesarsage share a test transcript?
- How would Prow look up the `deploy_id` for a PR's latest deploy preview? Likely `GET /sites/{site_id}/deploys?branch=deploy-preview-{pr_number}` — needs verification.
- Is relying on an undocumented Netlify endpoint acceptable for a feature in an upstream project? What happens when Netlify changes it without notice?
- Should Netlify credentials (PAT or build hook URL) be stored in a Kubernetes Secret and referenced from the trigger plugin config? What's the precedent for other external credentials in Prow?
