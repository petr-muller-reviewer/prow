---
pr: kubernetes-sigs/prow#1010
title: "config: allow loading job config from multiple paths"
head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
base: main
reviewed_at: 2026-10-22T12:38:44Z
verdict: request-changes
gate:
  decision: hold
  gated_at: 2026-10-21T12:02:56Z
  gated_head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
  reviewed_head_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
refresh_log:
  - old_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
    new_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
    summary: "No code changes; incorporated the symlink-root clarification and Oct 21–22 review activity, including the approval."
  - old_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
    new_sha: ac8ffb07e2ac7036c6bb4290f068a351508e7fad
    summary: "No code or comment activity; recorded that the PR was merged by app/kubernetes-prow."
---

# Review

## Gate

Hold: the operator migration and rollback documentation remains unaddressed.

The saved gate assessment was Hold on 2026-10-21 at head `ac8ffb07e2ac7036c6bb4290f068a351508e7fad`. The PR was merged by `app/kubernetes-prow` on 2026-10-22 at 11:27:45 UTC as merge commit `6c4d88e3b73e434d7acc03395ec337fa2b3f1fad`; the PR head did not change, and no comments or reviews arrived after the previous refresh. The hold and findings below remain the recorded pre-merge assessment. The operator migration and rollback guidance was still absent from the reviewed head.

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

This is the pre-merge assessment. The PR was subsequently merged on 2026-10-22 at 11:27:45 UTC.

## What this PR does

- Makes `--job-config-path` repeatable and changes the Go configuration APIs to accept path lists.
- Merges job configuration from files and recursively searched directories, with basename uniqueness across sources.
- Aggregates load errors and rejects the entire result on failure, using a separate `.prowignore` parser for each directory root.
- Extends config reload/watch handling and sinker size reporting to all configured paths and documents the CLI behavior.

Since previous review:

- No code changed: the PR head remains `ac8ffb07e2ac7036c6bb4290f068a351508e7fad`.
- On Oct 21–22, petr-muller and Prucek discussed path overlap, symlink behavior, and ConfigMap coupling. Prucek clarified that duplicate basenames fail but a directory symlink used as the root can silently load no files; petr-muller said the remaining path and ConfigMap observations were not blockers or in scope.
- On Oct 22 at 11:05 UTC, petr-muller canceled `/hold` and approved the PR; the Kubernetes Prow bot posted the approval notification.

Since the previous refresh (2026-10-22T11:12:22Z):

- No code changes: the PR head remains `ac8ffb07e2ac7036c6bb4290f068a351508e7fad`.
- The PR was merged by `app/kubernetes-prow` at 2026-10-22T11:27:45Z as `6c4d88e3b73e434d7acc03395ec337fa2b3f1fad`; no issue comments, inline comments, or reviews were added after the previous refresh.

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

### [should-fix] Reject directory symlink roots that load no configs
- where: `pkg/config/config.go:1748-1768`
- concern: `os.Stat` follows a symlink to a directory, but `filepath.Walk` does not follow a directory symlink at the root. The path therefore passes the directory check and can return success with an empty config, despite the documented promise to search directory paths recursively. Detect and report this case or walk the resolved directory. The author clarified this behavior in the review discussion; the maintainer considered the related path validation non-blocking.
- excerpt: |
    stat, err := os.Stat(jobConfig)
    if err != nil {
        r.errs = append(r.errs, err)
        return
    }
    if !stat.IsDir() {
        // still support a single file.
        r.readFile(jobConfig)
        return
    }
    r.prowIgnore, err = gitignore.NewRepositoryWithFile(jobConfig, ProwIgnoreFileName)
    if err != nil {
        r.errs = append(r.errs, fmt.Errorf("failed to create `%s` parser: %w", ProwIgnoreFileName, err))
        return
    }
    if err := walk(jobConfig, r.visit); err != nil {
        r.errs = append(r.errs, err)
    }

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

## Followups

Post-merge followups for kubernetes-sigs/prow#1010 — "config: allow loading job config from multiple paths", merged as `6c4d88e3b73e434d7acc03395ec337fa2b3f1fad`.

Decisions: **6 accepted** (5 deliver, including 1 design plan; 1 probe), **1 skipped**. The migration and rollback documentation followup was skipped; its existing review finding remains historical and is not marked resolved.

### 1. Reject symlinked directory roots that silently load no jobs

- Category: correctness
- Necessity: should
- Mode: deliver
- Where: `pkg/config/config.go:1747-1768`
- Why followup: The review discussion confirmed that a directory symlink root can silently load no jobs; the related validation work was treated as non-blocking and remained unfixed at merge.

```text
In kubernetes-sigs/prow, following merged PR #1010 — "config: allow loading job config from multiple paths" (merge commit 6c4d88e3b73e434d7acc03395ec337fa2b3f1fad), prevent directory symlink roots from silently loading no jobs. Work from the merged default branch in a fresh worktree, and inspect the current implementation before editing.

Mode: deliver

In pkg/config/config.go, jobConfigReader.readPath uses os.Stat, which follows a directory symlink, before passing the original path to filepath.Walk, which does not traverse a directory symlink root. Reject such roots with an actionable error naming the configured path. Document the restriction in the ReadJobConfigs API comment and site/content/en/docs/scaling.md.

Acceptance criteria: a directory symlink root containing valid job YAML fails explicitly; a mixed load with an ordinary valid source and a directory symlink root returns an error and an empty JobConfig, regardless of source order; ordinary directory roots, direct file sources, and symlinked YAML files used by ConfigMap mounts retain their behavior. Add focused regression tests and run the relevant config-package tests.

Scope guard: do not follow directory symlinks recursively, add general overlap validation, change basename or duplicate-job rules, or redesign reload/watch handling. Keep the fix limited to rejecting directory symlink roots and documenting that behavior.
```

### 2. Enforce the disjoint-path contract explicitly

- Category: validation
- Necessity: should
- Mode: deliver
- Where: `pkg/config/config.go:1700-1726; pkg/flagutil/config/config.go:60`
- Why followup: The merged documentation requires disjoint paths, but enforcement still depends on duplicate loaded basenames. The reviewer explicitly deferred a dedicated overlap check.

```text
In kubernetes-sigs/prow, following merged PR #1010 — "config: allow loading job config from multiple paths" (merge commit 6c4d88e3b73e434d7acc03395ec337fa2b3f1fad), enforce the documented disjoint-path contract independently of loaded YAML basenames. Work from the merged default branch in a fresh worktree, and inspect the current implementation before editing.

Mode: deliver

Add explicit validation at the ReadJobConfigs boundary in pkg/config/config.go. Reject repeated configured paths, overlapping ancestor/descendant directory roots, and an explicit file already contained in another configured directory. Normalize relative/absolute paths and cleaned spellings before comparing them, using path-component-aware containment rather than string prefixes. Report both original configured paths in an actionable error. Match the API comment, CLI help in pkg/flagutil/config/config.go, and site/content/en/docs/scaling.md to the enforced behavior.

Acceptance criteria: repeated roots, relative/absolute aliases, nested roots, and file/directory overlaps fail regardless of input order, whether roots are empty, and whether .prowignore excludes their contents. Disjoint siblings such as jobs and jobs-extra succeed. Failed combined reads return an empty JobConfig. Add focused tests and run relevant config-package tests. Preserve existing basename uniqueness, duplicate-job checks, and independent per-root ignore rules.

Scope guard: do not traverse directory symlinks, redesign filesystem identity or hard-link handling, relax ConfigMap filename constraints, or change reload/watch mechanisms. Coordinate with any directory-symlink-root rejection already present; keep this change focused on configured path overlap.
```

### 3. Prove all-or-nothing results and reload recovery

- Category: tests
- Necessity: should
- Mode: probe
- Where: `pkg/config/config_test.go:3730,3898-3908,3947-3959; pkg/config/agent.go:239-352,382-418`
- Why followup: The saved error-test finding remained unresolved, and the PR did not exercise running polling/watch agents through multi-source failure and recovery.

```text
In kubernetes-sigs/prow, following merged PR #1010 — "config: allow loading job config from multiple paths" (merge commit 6c4d88e3b73e434d7acc03395ec337fa2b3f1fad), probe the all-or-nothing loading and live reload behavior of multiple job configuration sources. Work from the merged default branch in a fresh worktree, and inspect current code and tests before proceeding.

Mode: probe

Write temporary tests in the config package or scratch space to exercise these behaviors: (1) ReadJobConfigs returns an empty JobConfig alongside read, YAML parse, merge, walk-callback, and walk-return errors, including failures after valid earlier files or sources were read; (2) a running polling agent retains the complete last accepted combined configuration when one source becomes invalid or missing; (3) watch mode retains that same configuration through a failed reload; (4) both modes resume publishing complete combined configurations after repair; (5) successful changes from either source are included, without silently dropping jobs from the other source. Use bounded waits and clean up watchers and goroutines where possible. Arrange source modifications so the separate known maximum-mtime masking problem does not obscure these transactional behavior checks.

For each failure, determine whether it is an implementation bug, an incorrect expectation, or a test/environment problem. A real bug means a failed combined load returns partial configuration, an agent publishes a partial/invalid snapshot or loses the last accepted valid snapshot, or recovery fails despite a valid combined configuration and an observed reload trigger. Apply only the minimal fixes for confirmed implementation bugs.

Acceptance criteria: exercise every named behavior; report evidence and classification for each failure; run relevant tests for any retained fix; delete all probe tests and temporary instrumentation before finishing. If no real bug is found, leave no files changed. Report the reproduction scenario for each confirmed bug so a maintainer can choose to retain regression tests separately.

Scope guard: the deliverable is a minimal bug fix, never the probe tests. Do not fix test-only mistakes, introduce a broad watcher lifecycle redesign, implement overlap/symlink validation, or redesign the polling timestamp algorithm. Do not change unrelated style or APIs.
```

### 4. Notice changes to each polled source independently

- Category: correctness
- Necessity: should
- Mode: deliver
- Where: `pkg/config/agent.go:355-418; pkg/config/agent_test.go:168-252`
- Why followup: Maximum-timestamp polling is inherited behavior; multiple independent sources make delayed detection more relevant. This improvement extends the merged capability.

```text
In kubernetes-sigs/prow, following merged PR #1010 — "config: allow loading job config from multiple paths" (merge commit 6c4d88e3b73e434d7acc03395ec337fa2b3f1fad), make polling notice changes to each configuration source independently. Work from the merged default branch in a fresh worktree, and inspect the current implementation before editing.

Mode: deliver

In pkg/config/agent.go, lastConfigModTime reduces the prow configuration and all job configuration roots to one maximum timestamp, and Agent.Start reloads only when that maximum increases, apart from the periodic fallback. Replace this decision with per-input modification-state tracking so a change to one source is noticed even when another source retains a later timestamp. Compare changes rather than requiring timestamps to increase. Keep the implementation bounded to the input metadata already observed by polling.

Acceptance criteria: add focused regression coverage showing that a source update is detected when its new timestamp remains below another input's timestamp; backwards timestamp changes are detected; changes to the prow config and every configured job root are considered; missing/invalid sources preserve the entire last accepted configuration and recovery allows complete updates again. Preserve initial-load error behavior, periodic fallback reloads, and combined-load validation. Use a small testable seam if needed to avoid waiting ten minutes, and run relevant config-package tests.

Scope guard: do not claim to solve recursive file-change detection, introduce full filesystem hashing, redesign fsnotify watchers or agent lifecycle, or implement overlap/symlink validation. Keep public API changes to the minimum needed, and preserve the existing fallback for changes that input metadata cannot reveal.
```

### 5. Make sinker metrics work with mixed configuration sources

- Category: deferred-review
- Necessity: could
- Mode: deliver
- Where: `cmd/sinker/main.go:574-710; cmd/sinker/main_test.go:1022-1130`
- Why followup: The reviewer deferred ConfigMap assumptions in monitoring. The maintainer selected a narrow fix preserving the existing ConfigMap-specific metric.

```text
In kubernetes-sigs/prow, following merged PR #1010 — "config: allow loading job config from multiple paths" (merge commit 6c4d88e3b73e434d7acc03395ec337fa2b3f1fad), make sinker's ConfigMap size monitoring tolerate ordinary configuration sources. Work from the merged default branch in a fresh worktree, and inspect the current implementation before editing.

Mode: deliver

Keep this followup narrow. In cmd/sinker/main.go, revise jobConfigMapMonitor/getConfigMapDirs discovery so configured ordinary files and non-ConfigMap directories are supported inputs without repeated monitoring errors. Discover actual ConfigMap mounts, including mounts nested among ordinary directories, and continue reporting their sizes. A flat ordinary source directory should remain an unmeasured source. Preserve contextual errors for genuine filesystem failures rather than treating every error as an unsupported source.

Acceptance criteria: add focused tests covering a direct ConfigMap mount, partitioned/nested mounts, a plain file source, flat and nested ordinary directories, and mixed ordinary/ConfigMap sources. Ordinary sources generate no ConfigMap-mount error noise; real mounts retain existing byte-size reporting and labels; genuine filesystem errors remain visible. Run the relevant sinker tests.

Scope guard: preserve the existing metric's name, labels, and ConfigMap-specific meaning. Do not add a general source-size metric, count ordinary checkout bytes as ConfigMap bytes, redesign configuration loading, relax basename constraints, or broaden this into a monitoring overhaul. This scope was explicitly narrowed during maintainer discussion.
```

### 6. Separate generic job loading from ConfigMap filename constraints

- Category: tech-debt
- Necessity: could
- Mode: deliver (design plan)
- Where: `pkg/config/config.go:1713-1720,1808-1814; pkg/plugins/updateconfig/updateconfig.go:280-370`
- Why followup: The reviewer deferred the generic reader's ConfigMap coupling. A design plan is needed before relaxing runtime constraints without weakening publication safety.

```text
In kubernetes-sigs/prow, following merged PR #1010 — "config: allow loading job config from multiple paths" (merge commit 6c4d88e3b73e434d7acc03395ec337fa2b3f1fad), produce a concrete plan to separate generic job loading from ConfigMap filename constraints. Work from the merged default branch in a fresh worktree and inspect current consumers before proposing changes.

Mode: deliver (design plan)

The generic reader in pkg/config/config.go requires globally unique YAML basenames because updateconfig can use basenames as ConfigMap keys. Trace that rule through runtime loading, checkconfig, pkg/plugins/updateconfig, and config-bootstrapper. Identify which checks protect job semantics and which protect publication to a particular ConfigMap. Account for explicit ConfigMap keys, UseFullPathAsKey, partitioned ConfigMaps, and independent sources with identical basenames. Determine where complete pre-publication validation can run; checking only one PR's changed files is insufficient if it misses collisions with existing published files.

Deliver CONFIG-SOURCE-VALIDATION-PLAN.md with a recommended bounded implementation, exact affected packages/functions, staged migration, compatibility considerations, and a focused test matrix. Acceptance criteria: the plan explains how independent sources can reuse basenames safely, how distinct files targeting the same ConfigMap key are rejected, how duplicate-job and overlapping-input checks remain enforced, and how CI/publishing safeguards land before any runtime constraint is relaxed. Distinguish confirmed current behavior from proposed behavior and unresolved questions.

Scope guard: deliver the plan before changing accepted configurations. Do not simply delete basename checks, modify runtime loading or publishing behavior, add a new CLI/API policy option without justification, redesign sinker metrics, or carry out the implementation.
```

