---
pr: kubernetes-sigs/prow#863
title: "announcement: notify addition of new /test-manual-required trigger command"
head_sha: 9ac7939fce9413c1548d0c1075e1f0d1ef28a18c
base: main
reviewed_at: 2026-09-01T11:18:38Z
verdict: request-changes
---

## What it does
- Adds an announcement for the `/test-manual-required` command in the `trigger` plugin.
- Places the August 20, 2026 entry before the older announcements.
- Describes which required presubmit jobs the command starts and which it excludes.

## Findings

### [should-fix] Replace the nonexistent `manual_trigger` configuration with the real selection criteria
- where: `site/content/en/docs/announcements.md:12-14`
- concern: `manual_trigger: required` is not a Prow configuration field, so the announcement may cause users to add invalid configuration or misunderstand how to opt a job into this command. The implementation selects required (`optional: false`) presubmits for which `NeedsExplicitTrigger()` is true — `always_run: false` (or omitted) with neither `run_if_changed` nor `skip_if_only_changed`. Describe those conditions in prose, or refer to the existing trigger-plugin documentation.
- excerpt: |
    required presubmits with `manual_trigger: required` that have no file-change conditions.

## Checked
- The announcement remains newest-first.
- `pkg/pjutil/filter.go:275-279` and its tests select required manually triggered jobs without change conditions, as the announcement otherwise states.
- This PR changes documentation only; no runtime or test changes are needed.

## Open questions
- None.
