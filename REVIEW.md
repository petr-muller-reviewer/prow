---
pr: kubernetes-sigs/prow#993
title: "Plugins: Support milestone_applier and repo_milestone in supplemental configs"
head_sha: 8992755740f1e716a5f4090faf94731f0244bfed
base: main
reviewed_at: 2026-10-08T11:20:06Z
verdict: needs-discussion
gate:
  decision: hold
  gated_at: 2026-10-07T11:37:54Z
  gated_head_sha: 8992755740f1e716a5f4090faf94731f0244bfed
  reviewed_head_sha: 8992755740f1e716a5f4090faf94731f0244bfed
refresh_log:
  - old_sha: c58fe94482f22969a7f18a39c97e57b3abf125a9
    new_sha: 8992755740f1e716a5f4090faf94731f0244bfed
    summary: "Added MilestoneApplier scope classification and coverage; resolved the scoped-config validation finding."
  - old_sha: 8992755740f1e716a5f4090faf94731f0244bfed
    new_sha: 8992755740f1e716a5f4090faf94731f0244bfed
    summary: "No Prow code changes; author linked follow-up PRs for Hook config loading and Kueue supplemental config."
---
# Review

## Gate

**Verdict: HOLD (recorded 2026-10-07)**

The prior scoped-config validation finding is resolved in the current head. The deployment wiring question has since been answered with tracked follow-up PRs: `kubernetes/k8s.io#10034` adds the supplemental-config flag, and `kubernetes/test-infra#37968` moves Kueue's config into a supplemental file. Both follow-up PRs remain open; this refresh records the new evidence but does not re-run the gate decision.

### Gating list

- **@mimowo inline comment, 2026-10-07** (`pkg/plugins/config.go:2356`): asks whether the Hook deployment mounts the supplemental directory and passes `--supplemental-plugin-config-dir=/etc/plugins`. On 2026-10-08 the author linked `kubernetes/k8s.io#10034`; its description says `/etc/plugins` is already mounted and its diff adds the flag. The question is answered and the deployment work is tracked, though feature rollout depends on those follow-ups.

### Area 2 — Independent merge risk

- No exported API, config schema, flag, or default is removed or changed. `milestone_applier` supplemental support is opt-in through the existing `--supplemental-plugin-config-dir` flag; installations that do not pass it are unaffected.
- When enabled, a supplemental `milestone_applier` entry changes plugin behavior for the configured org/repo, and duplicate keys fail configuration loading. This is scoped to deployments adding the new supplemental entries.
- No migration is required for existing configurations. The linked deployment/config follow-ups must land for the intended Kueue rollout.
- The `repo_milestone` title mismatch is non-gating: the PR body and author example specify `milestone_applier` only; the title should be narrowed.

## What this PR does

- Adds `milestone_applier` to the supplemental config allowlist.
- Merges per-repository branch-to-milestone maps and reports duplicate repository keys.
- Classifies milestone-applier org/repo keys for supplemental config hierarchy validation.
- Adds tests for merging, duplicate keys, and org/repo scope classification.

Since previous review:

- No code changed in `kubernetes-sigs/prow#993`; the head remains `8992755740f1e716a5f4090faf94731f0244bfed`.
- On Oct 8, the author linked `kubernetes/k8s.io#10034`, which adds `--supplemental-plugin-config-dir=/etc/plugins` and states the directory is already mounted; `kubernetes/test-infra#37968` handles syncing Kueue's supplemental file.

## Findings

### [nit] Align the PR title with the implemented field
- where: `pkg/plugins/config.go:2353-2356, 2387-2389`
- concern: The PR body and author’s Kueue example describe `milestone_applier`, while the title still advertises `repo_milestone`. That field remains outside the supplemental allowlist and merge path; narrow the title to avoid promising unsupported behavior.
- excerpt: |
    Lgtm: other.Lgtm, Plugins: other.Plugins, Triggers: other.Triggers, Welcome: other.Welcome,
    MilestoneApplier: other.MilestoneApplier},
    ...
    if err := c.mergeMilestoneApplierFrom(other.MilestoneApplier); err != nil {

### [nit] Consider sharing the duplicate-key map merge logic
- where: `pkg/plugins/config.go:2411-2425`
- concern: This repeats the map initialization, duplicate-key reporting, and insertion structure in `mergeExternalPluginsFrom` at lines 2394-2409. This is a judgement-call smell-baseline finding; a shared helper could keep these parallel merge paths consistent.
- excerpt: |
    func (c *Configuration) mergeMilestoneApplierFrom(other map[string]BranchToMilestone) error {
        if c.MilestoneApplier == nil && other != nil {
            c.MilestoneApplier = make(map[string]BranchToMilestone)
        }

        var errs []error
        for orgOrRepo, branchToMilestone := range other {
            if _, ok := c.MilestoneApplier[orgOrRepo]; ok {
                errs = append(errs, fmt.Errorf("found duplicate config for milestone_applier.%s", orgOrRepo))
                continue
            }
            c.MilestoneApplier[orgOrRepo] = branchToMilestone
        }

## Resolved

### Classify `milestone_applier` in supplemental scope validation

Resolved in `8992755740f1e716a5f4090faf94731f0244bfed`: `HasConfigFor` now includes `MilestoneApplier` in its config projection and classifies its org/repo keys. `TestHasConfigFor` now covers the field.

### Confirm Hook supplemental-config wiring

Answered by the author on 2026-10-08, who linked `kubernetes/k8s.io#10034`. That PR adds `--supplemental-plugin-config-dir=/etc/plugins` and states that `/etc/plugins` is already mounted. `kubernetes/test-infra#37968` handles moving/syncing Kueue's supplemental config. Both PRs remain open, so deployment of the feature depends on those follow-ups.

## Checked

- Existing main plugin configurations remain unaffected; the supplemental merge behavior is additive and needs no migration.
- Duplicate keys fail config loading, and reload errors retain the last successfully loaded configuration.
- The new tests cover `milestone_applier` merging and scope classification.
- No tests were run as part of this review.
