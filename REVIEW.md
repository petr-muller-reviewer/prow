---
pr: kubernetes-sigs/prow#769
title: "Add Git ref mutation methods to GitHub client"
head_sha: b733a7e10132a3567fb5e930312bb6287ff9831a
base: main
reviewed_at: 2026-06-24T11:35:20Z
verdict: approve
---

## Findings

### [should-fix] Ref format convention undocumented on UpdateRef
- where: `pkg/github/client.go:3476-3497`
- concern: CreateRef requires a fully-qualified ref (`refs/heads/my-branch`) while UpdateRef/GetRef take an unqualified ref (`heads/my-feature`). The difference is encoded in the URL path construction but not in the doc comments for UpdateRef/UpdateRefWithContext. Two independent reviewers flagged this as an easy source of caller mistakes.
- excerpt: |
    // UpdateRef updates a ref to the given SHA.
    func (c *client) UpdateRef(org, repo, ref, sha string, force bool) error {
    // UpdateRefWithContext updates a ref to the given SHA.
    func (c *client) UpdateRefWithContext(ctx context.Context, org, repo, ref, sha string, force bool) error {

### [nit] Displaced comment in fakegithub.go
- where: `pkg/github/fakegithub/fakegithub.go:106-113`
- concern: The comment `// A list of refs that got deleted via DeleteRef` now reads as if it documents `RefsCreated` because the new fields were inserted between the comment and `RefsDeleted`.
- excerpt: |
    // A list of refs that got deleted via DeleteRef
    RefsCreated []struct{ Org, Repo, Ref, SHA string }
    RefsUpdated []struct {
        Org, Repo, Ref, SHA string
        Force               bool
    }
    RefsDeleted []struct{ Org, Repo, Ref string }

### [nit] TestUpdateRef only tests force: false
- where: `pkg/github/client_test.go:678-698`
- concern: The test only exercises the `force: false` path. A table-driven case with `force: true` would verify the boolean serializes correctly rather than always being the zero value.

### [nit] CreateRefWithContext doc omits namespace requirement
- where: `pkg/github/client.go:3458`
- concern: CreateRef's doc says the ref must include its namespace (`refs/heads/my-branch`), but the WithContext variant omits this. Since WithContext is where the implementation lives, it should carry the full contract.

## Checked
- GetRef refactor to delegate to GetRefWithContext is semantically equivalent (c.request already calls requestWithContext with context.Background internally)
- Adding methods to CommitClient interface does not break other implementors: downstream code uses narrow per-plugin interfaces, not CommitClient directly
- map[string]string (CreateRef) vs map[string]interface{} (UpdateRef) difference is correct: force field must serialize as JSON boolean
- IsUnprocessableEntity follows same errors.As + status-code pattern as existing IsNotFound; intentionally lacks the ErrorMessages() fallback that IsNotFound has (that fallback is legacy)
- FakeClient implementations acquire mutex before appending to tracking slices
- Tests verify HTTP method, path, request body contents, and error classification (wrapped 422 recognized, 400 not)
- New mutating methods are dry-run safe (POST/PATCH short-circuited by requestRawWithContext)
- No new API token scopes required
- DeleteRef lacking a WithContext variant is a pre-existing asymmetry, not a blocker

## Open questions
- None
