---
pr: kubernetes-sigs/prow#739
title: "chore(deps): bump github.com/go-git/go-git/v5 from 5.19.0 to 5.19.1"
head_sha: 78b69904da08ed9005c4f63cbf48a8e6c610f463
base: main
reviewed_at: 2026-06-02T22:53:10Z
verdict: approve
---

## Summary

Security patch updating go-git from v5.19.0 to v5.19.1. Contains 14 fixes for shell injection, submodule handling, pack file validation, and DoS prevention. Published May 18 2026. Backward-compatible patch version. Unanimous approval from code quality, maintainability, and deployment risk perspectives.

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
