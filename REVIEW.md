---
pr: kubernetes-sigs/prow#1024
title: "peribolos: support delete_branch_on_merge repo setting"
head_sha: 3ad2362395c5d629b9880194d4ffe6df7e3428f5
base: main
reviewed_at: 2026-10-21T23:56:17Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-21T23:56:55Z
  gated_head_sha: 3ad2362395c5d629b9880194d4ffe6df7e3428f5
  reviewed_head_sha: 3ad2362395c5d629b9880194d4ffe6df7e3428f5
---

# Review

## Gate

**Verdict: merge**

The current PR head matches the head covered by the review, and there are no new commits, change requests, substantive inline comments, or blocking findings. The two saved findings are nonblocking test suggestions. The change adds optional behavior and does not alter existing deployments when the setting is unset.

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
