---
pr: kubernetes-sigs/prow#1010
title: "config: allow loading job config from multiple paths"
head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
base: main
reviewed_at: 2026-10-15T23:38:03Z
verdict: request-changes
gate:
  decision: hold
  gated_at: 2026-10-21T12:02:56Z
  gated_head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
  reviewed_head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
---

# Review

## Gate

Hold: the operator migration and rollback documentation remains unaddressed.

As of 2026-10-21, PR #1010 is open at reviewed head `ac8ffb07e2ac7036c6bb4290f068a351508e7fad`; there are no code changes since the saved review and no submitted GitHub reviews, inline comments, or human issue comments. The current checkout differs from the PR head only in review artifacts. The implementation assessment remains unchanged, but existing deployment arguments can break after an image upgrade, and rollback after adopting multiple paths can silently omit jobs. The operator documentation still does not explain these cases.

Prior findings disposition:

- **Not addressed; blocks merge:** **Document deployment migration and rollback** (`REVIEW.md`, `site/content/en/docs/scaling.md:71-75`). The scaling guide describes the new capability and path constraints but omits empty-value migration, repeated flags formerly used as overrides, and safe rollback. Add these instructions to the scaling guide and the breaking-change announcements to unblock this gate.
- **Not addressed; non-blocking:** **Clarify the disjoint-path contract** (`REVIEW.md`, `pkg/config/config.go:1700-1703`). The API still requires disjoint paths, while the reader detects overlaps only when loaded basenames collide (`pkg/config/config.go:1713-1726`, `1808-1814`). The original maintainer review explicitly treated this as optional; it does not independently gate merge.

Gating list:

- `site/content/en/docs/scaling.md:71-75` — source: saved maintainer review and direct compatibility assessment. Publish operator guidance to omit unused flags, remove earlier flag values from override wrappers, upgrade all consuming components before using multiple paths, and restore one aggregate path before rollback.

Independent merge risks:

- **Empty flag values:** `pkg/flagutil/config/config.go:60`, `pkg/flagutil/strings.go:63-69`, and `pkg/config/config.go:1748-1751`. An explicit empty flag is retained as a path and fails the initial load. This affects any component deployment or template emitting `--job-config-path=`; omitting the flag retains the previous no-separate-job-config behavior. The change is acknowledged in the PR body but lacks an operator migration announcement.
- **Repeated flags and rollback:** `pkg/flagutil/config/config.go:60`. Existing invocations with repeated flags switch from last-value-wins to loading every source, potentially restoring unintended jobs or failing on duplicates. Multiple sources are opt-in for deployments that previously used at most one value, but existing override wrappers are affected on upgrade. Once multiple paths are adopted, an older binary loads only the final path, so components must be upgraded before enabling the new arguments and arguments/mounts must be restored before rollback. This is not documented in the operator guide.
- **Go API compatibility:** `pkg/config/config.go:1661`, `1667`, `1708`; `pkg/config/agent.go:345`, `382`; `pkg/flagutil/config/config.go:35`. Downstream Go consumers must update renamed fields/functions and scalar-to-slice arguments before rebuilding. This deliberate source break is disclosed in the PR body and consistent with the library compatibility policy in `site/content/en/docs/overview/_index.md`, so it is a migration caveat rather than an additional merge blocker.
- No new Kubernetes API/CRD, job YAML schema, or access-control risk was identified. This gate used direct code and documentation analysis plus the saved maintainer perspectives; no additional listed compatibility skill applies. Tests were not run as part of this gate.

## Verdict

Request changes: document the deployment migration and rollback requirements before merging.

The three maintainer perspectives found the implementation cohesive, with low maintenance burden and no confirmed introduced correctness defects. Maintainability and Deployment Risk independently identified missing operator guidance; the confirmed changes to empty and repeated flag values, plus silent source omission by older binaries, require a migration notice. Deployment risk is medium. Tests were inspected but not run.

## What this PR does

- Makes `--job-config-path` repeatable and changes the Go configuration APIs to accept path lists.
- Merges job configuration from files and recursively searched directories, with basename uniqueness across sources.
- Aggregates load errors and rejects the entire result on failure, using a separate `.prowignore` parser for each directory root.
- Extends config reload/watch handling and sinker size reporting to all configured paths and documents the CLI behavior.

## Findings

### [blocking] Document deployment migration and rollback
- where: `site/content/en/docs/scaling.md:71-75`
- concern: Before merging, add guidance here and in the breaking-change announcements: operators must omit an unused `--job-config-path` because an explicit empty value now prevents startup, and wrappers that previously appended an override must account for every repeated value now being merged. Upgrade all consuming components before enabling multiple paths, and restore a single aggregate path before rollback because older binaries silently load only the final path. Maintainability and Deployment Risk both flagged the migration documentation gap; these compatibility changes were confirmed against the current and base implementations.
- excerpt: |
    `--job-config-path` can be passed multiple times when the job configs are not
    all in one place, for example when they are stored on different volumes. The
    job configs of all the paths are merged. A path is either a file or a directory
    that is searched recursively, the paths must not overlap, and the base names of
    the files must be unique across all the paths, not only within each of them.

### [should-fix] Clarify the disjoint-path contract
- where: `pkg/config/config.go:1700-1703`
- concern: Non-blocking maintainability suggestion: the contract requires disjoint paths, but the implementation rejects overlaps only when they yield duplicate loaded basenames. Empty or ignored overlapping roots can therefore succeed until their contents change; either explicitly validate overlap or describe the narrower enforced constraint.
- excerpt: |
    // ReadJobConfigs reads the JobConfig yaml from all the given paths and merges
    // it, but does not expand or validate the result. Every path is either a single
    // file or a directory that is searched recursively. File basenames must be
    // unique across all the paths, and the paths must not overlap.

### [nit] Assert an empty configuration after load errors
- where: `pkg/config/config_test.go:3907-3908`
- concern: Optional test improvement: before returning from error cases, assert that `actual` is an empty `JobConfig`, especially when valid files were read before another source failed. The implementation currently discards partial results correctly, but these assertions would not catch a regression that returned partial configuration alongside the expected error.
- excerpt: |
    if tc.expectedError != "" {
        return
    }

## Resolved

- Withdrawn: **Handle file paths in sinker ConfigMap size reporting**. The base revision already passes a file path to the same directory-oriented helper; this PR does not introduce the problem.
- Withdrawn: **Name the loaded-file counter**. This is a style suggestion without meaningful impact on correctness or maintainability.

## Checked

- Reviewed head `ac8ffb07e2ac7036c6bb4290f068a351508e7fad` against merge base `9851264902811315625eb653081f981dfb499289`; independent Code Quality and Maintainability reviewers both recommended approval, with no code blockers.
- The accumulator enforces basename uniqueness across files and directories, reports both conflicting paths, aggregates failures, and returns no partial job config on error.
- `.prowignore` parsing is scoped to each directory root.
- Repeated flag values retain their order; defaults are replaced using the existing flag utility. Omitting the flag and passing a single nonempty path retain their prior behavior.
- Config polling, watch setup, and sinker monitoring iterate over all configured paths. In-tree callers were migrated to the new Go API.
- Running agents retain their last accepted configuration after failed reloads. All sources share a validation boundary, so one invalid source blocks the combined update.
- Exported Go APIs intentionally change; downstream consumers must migrate, consistent with the repository's explicit library compatibility policy.
- No job YAML schema, Kubernetes API/CRD, RBAC, webhook, secret-handling, or external service changes were identified.
- Reviewed the added tests for repeated flags, initial loads, recursion, duplicate basenames/jobs, ignore rules, errors, and modification-time selection. Tests were not run; end-to-end multi-path reload/watch behavior remains untested by the added cases.
- Maximum-timestamp polling can delay updates whose timestamps remain below another input's timestamp; this is inherited behavior and was not treated as an introduced defect.

## Open questions

None.
