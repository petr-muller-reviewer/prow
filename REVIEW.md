---
pr: kubernetes-sigs/prow#863
title: "announcement: notify addition of new /test-manual-required trigger command"
head_sha: 30e93e59af1d4723818fbde35e72894e29f1d6d0
base: main
reviewed_at: 2026-09-27T14:53:44Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-27T14:54:14Z
  gated_head_sha: 30e93e59af1d4723818fbde35e72894e29f1d6d0
  reviewed_head_sha: 30e93e59af1d4723818fbde35e72894e29f1d6d0
refresh_log:
  - old_sha: 9ac7939fce9413c1548d0c1075e1f0d1ef28a18c
    new_sha: 30e93e59af1d4723818fbde35e72894e29f1d6d0
    summary: "Corrected announcement, resolved the sole finding, and cleared the superseded hold gate."
---

## Gate

**Verdict: merge.** At head `30e93e59a`, the announcement uses the actual presubmit selection criteria and no longer mentions the nonexistent `manual_trigger` field. The sole prior should-fix finding is addressed, @petr-muller approved the correction, and the PR changes documentation only.

**Gating list**
- None outstanding. The `REVIEW.md` should-fix finding at `site/content/en/docs/announcements.md:12-15` was resolved by `30e93e59a`.

**Independent merge risk**
- No notable merge risk. The full PR diff adds four lines to `site/content/en/docs/announcements.md`; it changes no API, configuration schema, default, or runtime behavior. Existing deployments are unaffected.

## What this PR does
- Adds an announcement for the `/test-manual-required` command in the `trigger` plugin.
- Places the August 20, 2026 entry before the older announcements.
- Describes which required presubmit jobs the command starts and which it excludes.

Since previous review:
- Commit `30e93e59a` replaced the nonexistent `manual_trigger: required` field with the actual job selection criteria in `site/content/en/docs/announcements.md:12-15` (4 additions, 3 deletions).
- @petr-muller approved the correction on 2026-09-27 at 14:53:25 UTC. The prior hold gate applied to `9ac7939` and is superseded by this revision.

## Findings

No open findings.

## Resolved

### [should-fix] Replace the nonexistent `manual_trigger` configuration with the real selection criteria
- where: `site/content/en/docs/announcements.md:12-14`
- concern: `manual_trigger: required` is not a Prow configuration field, so the announcement may cause users to add invalid configuration or misunderstand how to opt a job into this command. The implementation selects required (`optional: false`) presubmits for which `NeedsExplicitTrigger()` is true — `always_run: false` (or omitted) with neither `run_if_changed` nor `skip_if_only_changed`. Describe those conditions in prose, or refer to the existing trigger-plugin documentation.
- excerpt: |
    required presubmits with `manual_trigger: required` that have no file-change conditions.
- resolution: Commit `30e93e59a` removes the nonexistent field and correctly describes `always_run: false` (or unset), no file-change conditions, and exclusion of optional jobs.

## Checked
- The announcement remains newest-first.
- `pkg/pjutil/filter.go:275-279` and its tests select required manually triggered jobs without change conditions, as the announcement otherwise states.
- This PR changes documentation only; no runtime or test changes are needed.
- The revised announcement matches `TestManualRequiredFilter.ShouldRun` and `NeedsExplicitTrigger()`; the PR remained open at head `30e93e59a` when refreshed.

## Open questions
- None.

## Followups

### [could] Name `/test-manual-required` in the lasting job guide
- category: docs
- where: `site/content/en/docs/jobs.md:180-186`
- why followup: PR #863 announces the command, and PR #735 already added it to generated trigger-plugin help. The job guide explains manually triggered presubmits but does not name the command; this discoverability improvement does not affect the merge decision.
- handoff prompt:

```text
In kubernetes-sigs/prow, following PR #863, "announcement: notify addition of new /test-manual-required trigger command", update site/content/en/docs/jobs.md in the "Trigger Types" section to name /test-manual-required as a way to start all non-optional presubmit jobs that require an explicit trigger. Explain the selection criteria accurately: optional: false, always_run: false (or unset), and neither run_if_changed nor skip_if_only_changed set. Check the wording against pkg/pjutil/filter.go (TestManualRequiredFilter.ShouldRun) and pkg/config/jobs.go (NeedsExplicitTrigger). Acceptance: a reader can discover the command from the lasting job guide and understand which jobs it starts without mistaking "non-optional" for an unconditional Tide or branch-protection requirement. Keep the change to site/content/en/docs/jobs.md; do not change the command implementation, plugin help, announcement, or tests.
```
