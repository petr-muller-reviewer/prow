---
pr: kubernetes-sigs/prow#1024
title: "peribolos: support delete_branch_on_merge repo setting"
head_sha: 3ad2362395c5d629b9880194d4ffe6df7e3428f5
base: main
reviewed_at: 2026-11-19T15:40:38Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-21T23:56:55Z
  gated_head_sha: 3ad2362395c5d629b9880194d4ffe6df7e3428f5
  reviewed_head_sha: 3ad2362395c5d629b9880194d4ffe6df7e3428f5
refresh_log:
  - at: 2026-11-19T15:40:38Z
    old_sha: 3ad2362395c5d629b9880194d4ffe6df7e3428f5
    new_sha: 3ad2362395c5d629b9880194d4ffe6df7e3428f5
    summary: "No code changes; recorded petr-muller's approval and the merge of PR #1024."
---

# Review

## Gate

**Verdict: merge**

The current PR head matches the head covered by the review, and there are no new commits, change requests, substantive inline comments, or blocking findings. The two saved findings are nonblocking test suggestions. The change adds optional behavior and does not alter existing deployments when the setting is unset.

**Status update (2026-11-19):** the PR was approved and merged at the same reviewed head (`3ad2362`) on 2026-10-22T00:33:39Z; no code changed after this review.

**Gating list**

- None.

**Area 2 — Independent merge risk**

- **Configuration:** `delete_branch_on_merge` is an optional pointer field. The current config loader uses `yaml.Unmarshal` without strict unknown-field checking, so older binaries ignore the new key; existing configs remain valid.
- **Behavior:** setting the field to `true` opts configured repositories into GitHub's automatic deletion of merged head branches. This is intentional; no migration is required.
- **Go API:** the change adds fields to exported structs in `pkg/github` and `pkg/config/org`. Downstream Go source using unkeyed composite literals for these structs could require updates; keyed literals and deployed binaries are unaffected. This is a limited source-compatibility caveat, not a merge blocker.

No specialized repository compatibility skill was available or relevant; this pass assessed the config, exported types, and behavior directly.

## Verdict

Approve with suggestions. The three maintainer perspectives found no blocking correctness, maintainability, or deployment concern. They suggested a few focused tests for the new field's false update and GitHub request conversion paths.

## What this PR does

- Adds `delete_branch_on_merge` to per-repository Peribolos configuration.
- Carries the setting through GitHub repository reads and create/update requests.
- Dumps enabled values and prunes the GitHub default of `false`.
- Preserves existing behavior for repositories whose config leaves the field unset.

Since previous review:

- No code changes: `head_sha` is unchanged (`3ad2362` → `3ad2362`).
- petr-muller approved at 2026-10-21T23:58:37Z; Prow applied the `approved` label and the PR merged at 2026-10-22T00:33:39Z.
- No new issue comments, inline review comments, or reviews beyond that approval; both saved findings remain open, nonblocking test suggestions.

## Findings

### [nit] Cover disabling the setting in update delta tests
- where: `cmd/peribolos/main_test.go:4861-4870`
- concern: Add a case where the current value is `true` and the configured value is explicitly `false`. This would verify that Peribolos sends an explicit false update; current cases cover enabling and an unchanged true value.
- excerpt: |
    description: "unchanged delete branch on merge produces no delta",
    current: github.FullRepo{
        Repo:                github.Repo{Name: repoName},
        DeleteBranchOnMerge: true,
    },
    newState: org.Repo{
        DeleteBranchOnMerge: new(true),
    },

### [nit] Test GitHub request conversion for the new field
- where: `pkg/github/types.go:438`
- concern: Add `pkg/github` coverage that checks `delete_branch_on_merge` in serialized create/update requests and verifies `ToRepo` preserves the value. Peribolos tests cover delta construction, but not this request-model boundary.
- excerpt: |
    DeleteBranchOnMerge *bool `json:"delete_branch_on_merge,omitempty"`

## Checked

- Code Quality: the field is wired through dump, create, update, and request conversion; the pointer preserves unset versus explicitly false semantics.
- Maintainability: the implementation follows existing repo-setting helpers and adds little ongoing complexity.
- Deployment Risk: risk is low; existing configs need no migration, and only an explicit `true` changes merged-branch behavior.
- The reviewers and this review did not run tests.

## Open questions

None.

Standards: 0 findings; Spec: 0 findings. Maintainer perspectives: 2 nonblocking test suggestions, 0 blocking concerns.

## Followups

PR #1024 merged as `c4e8f2a912a4cb1c3eeab0cb73e653baeee74c6f`. The first three entries were recorded before the merge and remain valid post-merge; the last two were identified in the post-merge followup pass.

### Cover disabling `delete_branch_on_merge`

- Category: tests
- Mode: probe
- Where: `cmd/peribolos/main_test.go:4861`

```text
In kubernetes-sigs/prow, following PR #1024 — “peribolos: support delete_branch_on_merge repo setting” at head 3ad2362395c5d629b9880194d4ffe6df7e3428f5, add a focused regression test in cmd/peribolos/main_test.go.

Extend TestNewRepoUpdateRequest with a case where current.DeleteBranchOnMerge is true and newState.DeleteBranchOnMerge is false. Assert that newRepoUpdateRequest returns a RepoUpdateRequest whose RepoRequest.DeleteBranchOnMerge points to false.

Run go test ./cmd/peribolos -run TestNewRepoUpdateRequest. Do not change production behavior or unrelated tests.
```

### Exercise the GitHub request-model boundary

- Category: tests
- Mode: probe
- Where: `pkg/github/types.go:438`

```text
In kubernetes-sigs/prow, following PR #1024 — “peribolos: support delete_branch_on_merge repo setting” at head 3ad2362395c5d629b9880194d4ffe6df7e3428f5, add focused pkg/github tests for the new repository request field.

Cover RepoRequest.ToRepo preserving DeleteBranchOnMerge for true and false pointers, RepoRequest.Defined returning true when only DeleteBranchOnMerge is set, and JSON serialization of create/update requests including delete_branch_on_merge while omitting it when nil.

Run go test ./pkg/github. Do not change production behavior or unrelated tests.
```

### Document Peribolos repository settings

- Category: docs
- Mode: deliver
- Where: `site/content/en/docs/components/cli-tools/peribolos.md:18`

```text
In kubernetes-sigs/prow, following PR #1024 — “peribolos: support delete_branch_on_merge repo setting” at head 3ad2362395c5d629b9880194d4ffe6df7e3428f5, document the new repository setting in site/content/en/docs/components/cli-tools/peribolos.md.

Add a concise repository-settings example under the org configuration documentation showing a repo with delete_branch_on_merge: true. Explain that omitted repo fields are not managed, while explicitly setting delete_branch_on_merge: false disables automatic head-branch deletion. Match the existing documentation tone and do not attempt to exhaustively document every repository field.

No production code changes are expected.
```

### Fix `RepoRequest.Defined()` to include squash merge commit settings

- Category: bug
- Mode: deliver
- Where: `pkg/github/types.go:491`

```text
In kubernetes-sigs/prow, following PR #1024 — “peribolos: support delete_branch_on_merge repo setting” (merged as c4e8f2a912a4cb1c3eeab0cb73e653baeee74c6f), fix github.RepoRequest.Defined() so it reports squash merge commit settings.

RepoRequest.Defined() (pkg/github/types.go:491) ORs a hand-maintained list of pointer fields but omits SquashMergeCommitTitle and SquashMergeCommitMessage. cmd/peribolos/main.go:1453 only calls UpdateRepo when delta.Defined() is true, so a repo whose only desired change is squash_merge_commit_title / squash_merge_commit_message is silently not updated. Add both fields to Defined(). Add pkg/github tests proving Defined() is true when each is the only field set and false for a zero RepoRequest; prefer a table or reflection-based test over every RepoRequest field so future additions cannot be forgotten the same way.

Acceptance criteria: both fields are included in Defined(); `go test ./pkg/github ./cmd/peribolos` passes; the new test fails before the fix and passes after.

Scope guard: do not change dump behavior or PruneRepoDefaults (separate followup), do not alter RepoUpdateRequest field semantics, and do not touch unrelated fields or tests.
```

### Record squash merge commit settings in `--dump`

- Category: bug
- Mode: deliver
- Where: `cmd/peribolos/main.go:463`

```text
In kubernetes-sigs/prow, following PR #1024 — “peribolos: support delete_branch_on_merge repo setting” (merged as c4e8f2a912a4cb1c3eeab0cb73e653baeee74c6f), make peribolos --dump record squash merge commit settings.

dumpOrgConfig (cmd/peribolos/main.go:463) builds its org.Repo from the FullRepo returned by GetRepo but omits SquashMergeCommitTitle and SquashMergeCommitMessage, even though both FullRepo and org.Repo carry them and newRepoCreateRequest / newRepoUpdateRequest already send them. A dump therefore silently drops those settings. Add both fields to the dumpOrgConfig literal, and add GitHub's documented defaults ("PR_TITLE" for squash_merge_commit_title, "COMMIT_MESSAGES" for squash_merge_commit_message) to org.PruneRepoDefaults (pkg/config/org/org.go:287) so default values are omitted from dumps like other defaults.

Acceptance criteria: TestDumpOrgConfig shows non-default squash values round-tripping into the dumped config, TestPruneRepoDefaults shows the defaults are pruned and non-defaults kept, and `go test ./cmd/peribolos ./pkg/config/org` passes.

Scope guard: do not change create/update request logic or RepoRequest.Defined() (separate followup); do not change how other repo fields are dumped; no unrelated refactors.
```
