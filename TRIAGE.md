---
issue: kubernetes-sigs/prow#767
title: "Prow bot didn't check CLA in multiple repos"
state: closed
labels:
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:10:39Z
verdict: duplicate
---

## Findings

### [cause] panic in cla plugin's GitHub search call with apps auth
- detail: `handle()` in the `cla` plugin called `gc.FindIssues(query, sort, asc)` without an org. When the installation used GitHub Apps auth, the apps-auth round-tripper panicked with `BUG apps auth requested but empty org, please report this to the test-infra repo`, aborting the CLA-label sync for that webhook event and leaving PRs without an up-to-date `cncf-cla-yes`/`cncf-cla-no` label.
- evidence: `pkg/github/app_auth_roundtripper.go:156` (panic site); pre-fix call site was `pkg/plugins/cla/cla.go:105` (`gc.FindIssues(...)`, no org argument).

### [related-code] fixed call site
- where: `pkg/plugins/cla/cla.go:105`
- excerpt: |
    issues, err = gc.FindIssuesWithOrg(org, fmt.Sprintf("%s repo:%s/%s type:pr state:open", se.SHA, org, repo), "", false)

### [related-code] fixed interface method
- where: `pkg/plugins/cla/cla.go:73`
- excerpt: |
    FindIssuesWithOrg(org, query, sort string, asc bool) ([]github.Issue, error)

### [related-pr] fix a bug with cla and github apps auth
- ref: kubernetes-sigs/prow#764
- relevance: Merged 2026-06-20, one file / 2-line diff. Changed `gc.FindIssues(...)` to `gc.FindIssuesWithOrg(org, ...)` in `pkg/plugins/cla/cla.go`, threading the event's org through to the apps-auth round-tripper. This is the exact root-cause fix for #767; already present on `main` at triage time.

### [reproducibility] not reproducible on current main
- detail: The fix is already merged and verified against current `main`; `grep -rn "FindIssues(" pkg/plugins/cla/` finds no remaining org-less call sites. Issue predates the fix's rollout by only ~1-2 days (#764 merged 2026-06-20, issue filed 2026-06-22, confirmed fixed 2026-06-23), consistent with rollout lag rather than an unfixed bug.

## Checked

- `pkg/plugins/cla/cla.go` on current `main` (HEAD `e601a1ffafd7d8d3a781238a4c5f4233d6248f68`) uses `FindIssuesWithOrg`, matching PR #764's diff.
- No other call sites in `pkg/plugins/cla/` still reference the org-less `FindIssues` method.
- Issue thread: reporter (janetkuo) closed the issue on 2026-06-23 with "Fixed now"; commenter (Prucek) confirmed the fix was #764, acknowledged by the reporter with a link to internal Slack discussion.

## Next steps

- None. No maintainer action required — issue is closed and root-caused to an already-merged fix (#764).
- If the same panic signature (`BUG apps auth requested but empty org`) resurfaces in another plugin, point to #764 as the fix pattern (thread org through to any `FindIssues`-style GitHub search call under apps auth).

## Open questions

- None.
