---
pr: 616
title: "Added SRC_BASE And SRC_HOST PodUtils Environment Variables"
head_sha: 8114f817c4502109aa375ec902531f7db272931d
base: main
reviewed_at: 2026-06-22
verdict: approve-with-suggestions
---

# PR #616: Added SRC_BASE And SRC_HOST PodUtils Environment Variables

[#616](https://github.com/kubernetes-sigs/prow/pull/616) by @midnightconman · Fixes [#302](https://github.com/kubernetes-sigs/prow/issues/302) · Merged · +145 −1 across 13 files

| Field | Value |
|---|---|
| Status | **merged** — lgtm + approved |
| Labels | area/pod-utilities, size/L |
| Reviewers | @Prucek (approved), @matthyx (lgtm) |
| Core change | `pkg/pod-utils/downwardapi/jobspec.go` |
| Test changes | 1 unit test file + 10 golden YAML fixtures + 1 pipeline controller test |

## What It Does

Adds two new environment variables to Prow job pods:

- `SRC_HOST` — the hosting service (e.g. `github.com`, `gitlab.com`)
- `SRC_BASE` — the path under the host (e.g. `org/repo`)

These allow job scripts to construct full source paths for `extra_refs` instead of using fragile relative paths.

## Resolution Logic

```
if Refs.PathAlias != "" {
  // e.g. PathAlias = "k8s.io/repo-name"
  SRC_HOST = PathAlias[:firstSlash]   // "k8s.io"
  SRC_BASE = PathAlias[firstSlash+1:] // "repo-name"
} else if Refs.RepoLink != "" {
  // e.g. RepoLink = "https://gitlab.com/org/repo"
  SRC_HOST = afterScheme[:firstSlash]   // "gitlab.com"
  SRC_BASE = afterScheme[firstSlash+1:] // "org/repo"
} else {
  SRC_HOST = "github.com"
  SRC_BASE = Refs.Org + "/" + Refs.Repo
}
```

Mirrors `clone.PathForRefs()` at `pkg/pod-utils/clone/clone.go:139`.

## Issues Found

### [minor] SRC_BASE = "/" when Org and Repo are empty

When `spec.Refs.Org` and `spec.Refs.Repo` are both empty (no PathAlias/RepoLink), `fmt.Sprintf("%s/%s", "", "")` produces `"/"`. Visible in the pipeline controller test at `controller_test.go:1189`. A consumer could misinterpret this as a filesystem root.

*`jobspec.go:150`*

### [moderate] New dependency on `pkg/github`

The `downwardapi` package previously had no dependency on `pkg/github`. The import is added solely for `github.DefaultHost` (the string `"github.com"`). This couples pod-utils/downwardapi to the GitHub client package, which could create import cycle risks as the codebase evolves. A local constant or a lighter-weight import would be preferable.

*`jobspec.go:27`*

### [minor] Naming: SRC_BASE semantics mismatch

The PR description says SRC_BASE should be `GOPATH + "/src"`, but the implementation sets it to the repo path under the host (e.g. `org/repo`). The name `SRC_BASE` suggests a base directory; something like `SRC_PATH` or `REPO_PATH` would better describe the actual value.

### [note] No documentation update

Existing env vars like `REPO_OWNER`, `REPO_NAME` are documented for job authors. The new variables should be documented so users know they exist and what values to expect.

## Test Coverage

**What's covered:**
- Default case (no PathAlias/RepoLink): SRC_HOST=`github.com`, SRC_BASE=`org/repo`
- PathAlias case (`"k8s.io/repo-name"`): SRC_HOST=`k8s.io`, SRC_BASE=`repo-name`
- RepoLink case (`"https://gitlab.com/org/repo"`): SRC_HOST=`gitlab.com`, SRC_BASE=`org/repo`
- All 10 golden YAML fixtures updated consistently
- Pipeline controller test updated

**Missing test cases:**
- `PathAlias` with no `/` (e.g. `"k8s.io"`) — sets SRC_HOST to entire alias, SRC_BASE stays as default `org/repo`
- `RepoLink` without a scheme (e.g. `"gitlab.com/org/repo"`) — works correctly but isn't explicitly tested
- Empty Org+Repo with no PathAlias/RepoLink — produces `SRC_BASE="/"`

## Changed Files

- `pkg/pod-utils/downwardapi/jobspec.go` — core logic + constants
- `pkg/pod-utils/downwardapi/jobspec_test.go` — 3 existing test updates + 2 new test cases
- `cmd/pipeline/controller_test.go` — pipeline params updated
- `pkg/pod-utils/decorate/testdata/TestProwJobToPod_{0-10}.yaml` — 10 golden fixtures

## Maintainer Review

### Code Quality: APPROVE

No critical issues found. Correctly mirrors `clone.PathForRefs` structure and precedence order. Uses `github.DefaultHost` rather than duplicating the string literal. Constants well-named and placed alongside related existing constants. Good test coverage across three main branches (default, PathAlias, RepoLink).

**Suggestion:** Add a test for PathAlias without `/` to document the edge case behavior.

### Maintainability: REQUEST_CHANGES

- **Duplicated logic**: `EnvForSpec` and `PathForRefs` must be kept in sync manually. A shared helper would eliminate this risk.
- **`SRC_BASE` naming misleading**: PR description says `GOPATH+"/src"`, code sets it to `org/repo`. The name suggests a filesystem path. Since this becomes a permanent public API, `SRC_PATH` or `SRC_REPO_PATH` would be clearer.
- **New `pkg/github` dependency** in `downwardapi` for a single constant. Increases coupling surface.
- **PathAlias-without-slash** demonstrates exactly the drift risk of duplicated logic — subtly different semantics from `PathForRefs`.

**Maintenance burden:** MEDIUM — duplicated logic that must stay in sync + naming that becomes permanent API.

### Deployment Risk: LOW

- Purely additive — no existing configuration, API, or behavior modified
- Every non-periodic job pod gets two new env vars; jobs not referencing them are unaffected
- Potential env collision if a job already defines `SRC_BASE`/`SRC_HOST` (same pre-existing pattern as `REPO_OWNER` etc.)
- Upgrade/rollback is clean: no config migration, no ordering dependencies
- Should be announced in release notes

### Converging Concerns (flagged by 2+ reviewers)

- **PathAlias-without-slash edge case** (Code Quality + Maintainability): When `PathAlias` has no `/` (e.g. `"k8s.io"`), `srcHost` gets the whole alias but `srcBase` retains default `org/repo`, producing `k8s.io/org/repo` instead of `k8s.io` as `PathForRefs` would. A test case should be added to document this behavior.
- **Duplicated logic with `clone.PathForRefs`** (Code Quality + Maintainability): The resolution logic is duplicated across two files with only a comment linking them. Both reviewers noted this as a consistency/drift risk. Extracting a shared helper like `clone.HostAndPathForRefs()` would make the relationship compiler-enforced.

## Final Recommendation

**Approve with suggestions.** This is a small, purely additive change that adds two new environment variables following established patterns. Deployment risk is low and the code is correct for all practical cases. The maintainability reviewer's REQUEST_CHANGES verdict raises valid long-term concerns about duplicated logic and naming, but these are not severe enough to block a change of this scope — the duplication is small (~10 lines), the naming is consistent with existing `SRC_` prefix patterns, and refactoring can happen incrementally.

### Suggestions

- **Add a test for PathAlias without `/`**: Independently flagged by two reviewers; low cost, high value.
- **Extract a shared helper in a follow-up**: A function like `clone.HostAndPathForRefs()` would eliminate the sync risk between `EnvForSpec` and `PathForRefs`. Better as a separate PR.
- **Consider `SRC_BASE` naming clarity**: Could be confused with a filesystem path. Since there are no existing users yet, a rename is still cheap.
- **Add a release note**: New env vars available to all non-periodic jobs should be announced.

### Draft PR Comment

> This looks good to merge. The change is purely additive, low-risk, and correctly mirrors the existing `PathForRefs` logic. Two reviewers independently flagged the PathAlias-without-slash edge case — I'd appreciate a test case covering that scenario before or shortly after merge. Longer term, extracting a shared helper to keep the path-resolution logic in one place would be a nice follow-up. Consider adding a release note for the two new environment variables.

## Review Checklist

- [ ] Confirm SRC_BASE="/" edge case is acceptable or needs guarding
- [ ] Evaluate `pkg/github` import — could it be replaced with a local const?
- [ ] Verify naming aligns with how consumers will use these vars
- [ ] Check if user-facing docs need an update
- [ ] Consider adding tests for PathAlias-without-slash and RepoLink-without-scheme
- [ ] Consider extracting shared helper to eliminate logic duplication
- [ ] Add release note for new environment variables

## Followups

### 1. docs: Add SRC_HOST and SRC_BASE to env var documentation
- **Category:** docs | **Necessity:** should | **Where:** `site/content/en/docs/jobs.md:336`

```
In `kubernetes-sigs/prow`, following the merge of PR #616 "Added SRC_BASE And SRC_HOST PodUtils Environment Variables" (merge commit 1ec9888010fc6f9e69b44aed1121402b7bb2e0ab):

The env var table in `site/content/en/docs/jobs.md` documents every pod-utils environment variable exposed to job pods (`REPO_OWNER`, `REPO_NAME`, `PULL_BASE_REF`, etc.) but the two new variables added by PR #616 are missing.

Add two rows to the table (around line 337, after `REPO_NAME`) for:

- `SRC_HOST` — available for postsubmit, batch, and presubmit jobs (not periodic). Description: "Hosting service for the repo source." Example: `github.com`. For PathAlias repos this is the host portion of the alias (e.g. `k8s.io`); for RepoLink repos it is the host from the link (e.g. `gitlab.com`).
- `SRC_BASE` — same availability. Description: "Path under the host for the repo source." Example: `kubernetes/test-infra`. For PathAlias repos this is the path portion after the host; for RepoLink repos it is the path portion after the host.

Follow the existing table format exactly. The descriptions should reflect the actual implementation (host/path splitting), NOT the PR description's claim about GOPATH+"/src".

Acceptance criteria: the two new rows appear in the table with correct availability checkmarks (not periodic), descriptions, and examples. No other changes.

Out of scope: changing variable names, adding release notes, modifying code.
```

### 2. tests: Add PathAlias-without-slash edge case test
- **Category:** tests | **Necessity:** should | **Where:** `pkg/pod-utils/downwardapi/jobspec_test.go`

```
In `kubernetes-sigs/prow`, following the merge of PR #616 "Added SRC_BASE And SRC_HOST PodUtils Environment Variables" (merge commit 1ec9888010fc6f9e69b44aed1121402b7bb2e0ab):

The `EnvForSpec` function in `pkg/pod-utils/downwardapi/jobspec.go` (lines 152-157) handles `PathAlias` by splitting on the first `/` to separate host from path. When `PathAlias` has no `/` (e.g. `"k8s.io"`), `srcHost` gets the entire alias but `srcBase` retains the default `org/repo` value. This differs subtly from `clone.PathForRefs` in `pkg/pod-utils/clone/clone.go:139-151`, which uses the entire `PathAlias` as the clone path without splitting.

Add one test case to `TestEnvironmentForSpec` in `pkg/pod-utils/downwardapi/jobspec_test.go` that exercises this edge case:
- Set `PathAlias` to `"k8s.io"` (no slash), with `Org: "org-name"`, `Repo: "repo-name"`.
- Assert that `SRC_HOST` is `"k8s.io"` and `SRC_BASE` is `"org-name/repo-name"`.
- Name the test case something like `"postsubmit job with path alias without slash"`.
- Follow the existing test case structure (see the `"postsubmit job with path alias"` case for the pattern).

Acceptance criteria: `go test ./pkg/pod-utils/downwardapi/...` passes with the new test case. The test documents the current behavior for this edge case.

Out of scope: changing the behavior, refactoring the resolution logic, modifying `PathForRefs`.
```

### 3. tech-debt: Extract shared host/path resolution helper
- **Category:** tech-debt | **Necessity:** should | **Where:** `pkg/pod-utils/clone/clone.go` + `pkg/pod-utils/downwardapi/jobspec.go`

```
In `kubernetes-sigs/prow`, following the merge of PR #616 "Added SRC_BASE And SRC_HOST PodUtils Environment Variables" (merge commit 1ec9888010fc6f9e69b44aed1121402b7bb2e0ab):

Two functions implement the same three-branch resolution logic (PathAlias > RepoLink > default github.com) independently:
- `EnvForSpec` in `pkg/pod-utils/downwardapi/jobspec.go` (lines 145-168) — splits into host and base path for SRC_HOST/SRC_BASE env vars.
- `PathForRefs` in `pkg/pod-utils/clone/clone.go` (lines 139-151) — constructs the full clone path.

They already diverge subtly: `PathForRefs` uses the entire `PathAlias` as a clone path, while `EnvForSpec` splits it on `/` to extract host vs base. Both import `github.DefaultHost` from `pkg/github` for the default.

Extract a shared function in `pkg/pod-utils/clone/`:
```go
func HostAndPathForRefs(refs prowapi.Refs) (host, basePath string)
```

This function should implement the three-branch resolution and return the host and path components separately. Then:
1. Refactor `PathForRefs` to call `HostAndPathForRefs` and join the results with `baseDir/src/host/basePath`.
2. Refactor `EnvForSpec` to call `clone.HostAndPathForRefs` for the `SRC_HOST` and `SRC_BASE` values, removing its inline resolution logic.
3. This refactor should also remove the `pkg/github` import from `downwardapi/jobspec.go` — the `DefaultHost` reference moves into `clone/` where it already exists.

Decide on the correct behavior for PathAlias-without-slash: either both callers should split (current EnvForSpec behavior) or both should use the whole alias (current PathForRefs behavior). Choose whichever is more correct and make the shared function implement it consistently.

Acceptance criteria: `go test ./pkg/pod-utils/...` and `go test ./cmd/pipeline/...` pass. `PathForRefs` and `EnvForSpec` both delegate to the shared function. The `pkg/github` import is removed from `downwardapi/jobspec.go`. No behavioral change for the common cases (PathAlias with `/`, RepoLink with scheme, default).

Out of scope: renaming the env vars, adding new env vars, changing documentation.
```

### 4. cleanup: Remove pkg/github import from downwardapi
- **Category:** cleanup | **Necessity:** could | **Where:** `pkg/pod-utils/downwardapi/jobspec.go:30`

```
In `kubernetes-sigs/prow`, following the merge of PR #616 "Added SRC_BASE And SRC_HOST PodUtils Environment Variables" (merge commit 1ec9888010fc6f9e69b44aed1121402b7bb2e0ab):

NOTE: If followup #3 (extract shared helper) is being done, skip this — it resolves the import naturally.

The `downwardapi` package (`pkg/pod-utils/downwardapi/jobspec.go`, line 30) imports `sigs.k8s.io/prow/pkg/github` solely for the constant `github.DefaultHost` (the string `"github.com"`, defined in `pkg/github/types.go:50`). This couples the pod-utils/downwardapi package to the GitHub client package unnecessarily.

Replace the `github.DefaultHost` reference at `jobspec.go:149` with a local constant (e.g. `const defaultGitHubHost = "github.com"`) and remove the `pkg/github` import. The `pkg/pod-utils/clone` import on line 31 already exists and is unrelated — keep it.

Acceptance criteria: `go test ./pkg/pod-utils/downwardapi/...` passes. The `pkg/github` import is removed. `go vet ./pkg/pod-utils/downwardapi/...` passes.

Out of scope: refactoring the resolution logic, changing behavior, touching clone.go.
```
