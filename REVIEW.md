---
pr: kubernetes-sigs/prow#739
title: "chore(deps): bump github.com/go-git/go-git/v5 from 5.19.0 to 5.19.1"
head_sha: 78b69904da08ed9005c4f63cbf48a8e6c610f463
base: main
reviewed_at: 2026-09-28T20:42:21Z
verdict: approve
refresh_log:
  - old_sha: 78b69904da08ed9005c4f63cbf48a8e6c610f463
    new_sha: 78b69904da08ed9005c4f63cbf48a8e6c610f463
    summary: "No code changes; approval recorded and PR merged."
---

## What this PR does

Security patch updating go-git from v5.19.0 to v5.19.1. Contains 14 fixes for shell injection, submodule handling, pack file validation, and DoS prevention. Published May 18 2026. Backward-compatible patch version. Unanimous approval from code quality, maintainability, and deployment risk perspectives.

Since previous review:

- No commits or file changes; the PR head remains `78b69904da08ed9005c4f63cbf48a8e6c610f463`.
- petr-muller approved the PR on 2026-06-02 at 23:01:40 UTC; k8s-ci-robot reported approval at 23:01:48 UTC. No new inline comments were added.
- k8s-ci-robot merged the PR on 2026-06-02 at 23:33:44 UTC.

## Findings

No findings. Clean dependency update with no code changes required.

## Checked

- Changes limited to `go.mod` and `go.sum`
- Patch version bump maintains backward compatibility per semver
- 14 documented security fixes in upstream release
- No prow code changes required
- CI labels present (ok-to-test, cncf-cla: yes)
- go-git usage limited to 6 files: testfreeze checker, in-repo config loading, test utilities
- go-git properly abstracted behind `checker` interface and `pkg/git/v2` wrapper
- Test coverage uses mocks, insulated from library internals
- Release published 2 weeks ago, giving time for upstream issues to surface
- Maintenance burden: LOW (reduces security debt, zero added complexity)
- Deployment risk: LOW (no config changes, no API migrations, drop-in replacement)

## Open questions

- Does Prow use submodule functionality with go-git? (stricter submodule name validation added upstream)

## Deployment notes

- Monitor logs post-deploy for new git-related errors when processing in-repo configs (`.prow.yaml`). Stricter validation rejects previously-tolerated malformed Git data.

## Dependency followups

No improvement opportunities identified. Examined the direct `github.com/go-git/go-git/v5` bump from v5.19.0 to v5.19.1; the PR changed no transitive module versions. The release adds security and correctness fixes, primarily in submodule handling, Git object parsing, SSH transport, and worktree path validation. Prow's production call in `pkg/plugins/testfreeze/checker/checker.go:162` lists remote refs; `test/integration/internal/fakegitserver/fakegitserver.go:188` initializes test repositories. Neither uses a deprecated API, a newly superseded API, or a new feature that would simplify these call sites.
