---
pr: kubernetes-sigs/prow#773
title: "ghproxy: support git ref paths with slash-separated branch names"
head_sha: 69b3da670e52cf3e5b6fe515eff76dc7ac27dcee
base: main
reviewed_at: 2026-06-30T17:47:47Z
verdict: approve
---

## What this PR does

- Fixes ghproxy's path simplifier to recognize git ref paths where branch names contain forward slashes (e.g. `feature/v1beta1-migrate`, `release/v2.0`)
- Changes `refs/heads/:ref` tree node from non-greedy `v("ref")` to `VGreedy("ref")` so all remaining path segments after `heads/` are consumed and represented as `:ref`
- Follows the same pattern already used for label paths at line 40
- Adds integration tests for both `/repos/:owner/:repo/...` and `/repositories/:repoId/...` path prefixes

## Findings

None. All three reviewer perspectives (code quality, maintainability, deployment risk) approved without identifying any blocking issues, improvements needed, or concerns.

## Checked

- **Code correctness**: Minimal surgical fix changing only one function call, correctly using `VGreedy` which matches all remaining path segments
- **Pattern consistency**: Follows existing precedent for labels (line 40) which also contain slashes
- **Test coverage**: Two new test cases covering both URL forms with realistic slash-containing branch names
- **Root cause addressed**: Previous `v("ref")` matched only single segment, causing multi-slash branches to trigger "Path not handled" debug logs
- **Maintenance burden**: LOW — one-character code change, no new abstractions, no cascading changes
- **Debuggability improvement**: Eliminates spurious debug log noise during incident investigation
- **Deployment risk**: LOW — purely internal metrics labeling fix, no config/API/ProwJob behavior changes
- **Backward compatibility**: Zero-downtime upgrade, no configuration migration needed, can be rolled back safely
- **Metrics impact**: Paths with slash-containing branches shift from `unmatched` metric series to correctly-labeled series

## Open questions

None.
