---
pr: kubernetes-sigs/prow#764
title: "fix a bug with cla and github apps auth"
head_sha: bf3efc64291622e53c18aabb17e83c238331c951
base: main
reviewed_at: 2026-06-23T20:22:45Z
verdict: approve
---

## Findings

### [nit] FindIssues wrapper is a footgun
- where: `pkg/github/client.go:3616-3621`
- concern: `FindIssues` has a comment warning it doesn't work with GitHub Apps auth but still exists as a public method. A follow-up to deprecate or remove it would prevent this bug class from recurring. Out of scope for this PR.

## Checked
- `org` value correctness: `se.Repo.Owner.Login` extracted on line 98, used by all other API calls in the same function
- Interface consistency: local `gitHubClient` interface (line 69) and call site (line 105) both updated to `FindIssuesWithOrg`
- Test coverage: `fakegithub.FakeClient` satisfies updated interface, tests pass without modification
- Pattern consistency: blunderbuss, trigger, welcome plugins all use `FindIssuesWithOrg` with org from event payloads
- `handleComment` path uses `GetCombinedStatus`, not affected by this bug class
- No remaining `FindIssues` callers in any plugin
- No config, CLI, API, or deployment manifest changes

## Open questions
(none)
