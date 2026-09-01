---
pr: kubernetes-sigs/prow#863
title: "announcement: notify addition of new /test-manual-required trigger command"
head_sha: 9ac7939fce9413c1548d0c1075e1f0d1ef28a18c
base: main
reviewed_at: 2026-08-20T13:18:49Z
verdict: approve
---

## What it does
- Adds a 3-line entry to `site/content/en/docs/announcements.md` announcing the `/test-manual-required` trigger-plugin command.
- Placed newest-first, above the August 11 entry, matching the file's existing ordering convention.

## Findings

### [question] "manual_trigger: required" not a literal config field
- where: `site/content/en/docs/announcements.md:12-14`
- concern: The phrase "with `manual_trigger: required`" reads like a literal plugins.yaml/presubmit field, but the actual behavior in `pkg/pjutil/filter.go` (`TestManualRequiredFilter.ShouldRun`) derives from `NeedsExplicitTrigger()`/`AlwaysRun`/`RunIfChanged`, not a field with that name. Likely just loose phrasing describing a concept, not a factual error worth blocking on.
- excerpt: |
    - *August 20, 2026* New `/test-manual-required` command in the `trigger` plugin. Triggers all
        required presubmits with `manual_trigger: required` that have no file-change conditions.
        Optional, automatically-triggered, and file-change-conditional jobs are excluded.

## Checked
- Newest-first ordering of announcements is preserved.
- Description matches actual filter logic in `pkg/pjutil/filter.go` (`TestManualRequiredFilter.ShouldRun`).
- No code changes in this PR; doc-only.

## Open questions
- Is "manual_trigger: required" meant as a literal config key/value, or just descriptive prose? If literal, it doesn't match current config field names.
