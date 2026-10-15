---
pr: kubernetes-sigs/prow#1010
title: "config: allow loading job config from multiple paths"
head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
base: main
reviewed_at: 2026-10-15T23:38:03Z
verdict: request-changes
gate:
  decision: hold
  gated_at: 2026-10-15T23:17:37Z
  gated_head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
  reviewed_head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
---

# Review

## Gate

Hold: document the deployment migration for changed `--job-config-path` behavior before merging.

The PR is open at the reviewed head `ac8ffb07e2ac7036c6bb4290f068a351508e7fad`, with no subsequent commits and no submitted GitHub reviews, inline comments, or human issue comments. The previous local findings were withdrawn for the reasons recorded below. The code review found no introduced defects, but the intentional flag changes affect existing deployments and are described only in the PR body; the checked-in documentation does not explain the migration.

Gating list:

- **Deployment migration notice** — `pkg/flagutil/config/config.go:60` and `pkg/config/config.go:1748-1751` (independent compatibility assessment). An explicit `--job-config-path=` previously selected no separate job config; it now stores an empty path, attempts `os.Stat("")`, and makes the initial config-agent load fail. Any component whose deployment template emits an empty flag can fail to start after updating its image. Add a breaking-change announcement telling operators to omit the flag when unused, or preserve the previous empty-value behavior.
- Include repeated-flag behavior in the same notice: `pkg/flagutil/config/config.go:60` changes repeated flags from last-value-wins to merging every value. Existing deployment wrappers that append an override must remove the old value; otherwise they may load unintended jobs or fail on duplicate basenames.

Independent merge risks:

- **Go source compatibility** — `pkg/config/config.go:1661`, `pkg/config/config.go:1667`, `pkg/config/config.go:1708`, `pkg/config/agent.go:345`, `pkg/config/agent.go:382`, and `pkg/flagutil/config/config.go:35` rename exported APIs or change their argument/field types. Downstream Go consumers must migrate to slices and the new names. This is deliberate and disclosed in the PR body; the library policy in `site/content/en/docs/overview/_index.md` explicitly disclaims backward compatibility, so it does not independently block merge.
- **Runtime compatibility** — the empty-value and repeated-value changes above affect existing invocations without opting into the new multi-path feature. The default with the flag omitted still means no separate job config. The announcements page promises documentation for configuration/deployment breaking changes, and this PR does not add such an entry.
- No Kubernetes API/CRD or job YAML schema changes were found. No additional repository compatibility skill applies to this diff; this assessment used the code, the PR discussion endpoints, and the documented library and announcement policies. Tests were not run as part of this gate.

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
