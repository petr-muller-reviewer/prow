---
pr: kubernetes-sigs/prow#809
title: "Add dependabot config for dep grouping"
head_sha: 65d166901b7a01f8958335d75d7902e88933472b
base: main
reviewed_at: 2026-07-27T22:40:31Z
verdict: approve
---

## Summary

Adds `.github/dependabot.yaml`: weekly `gomod` version updates for the root
module, with groups (`kubernetes`, `prometheus`, `aws`, `golang-x`) so
correlated dependency bumps land as one PR instead of several conflicting
ones. No dependabot config existed before this PR — prior automated bump PRs
(#806, #807) came from Dependabot security alerts, which don't require
config and scan all go.mod files regardless.

## Findings

### [should-fix] hack/tools/go.mod not covered by grouping
- where: `.github/dependabot.yaml:1-20`
- concern: The repo has three Go modules (`/`, `/hack/tools`, `/site`). Only `directory: "/"` is configured. `hack/tools/go.mod` vendors `k8s.io/code-generator`, `sigs.k8s.io/*`, and many `github.com/aws/aws-sdk-go-v2/*` indirect deps — the exact families this PR groups. The two PRs merged immediately before this one (#806, #807) were separate `golang.org/x/crypto` bumps in `/` and `/hack/tools` at the same time — precisely the churn/duplicate-review scenario the PR aims to fix, and it will still recur for `hack/tools` since it's not configured.
- excerpt: |
    version: 2
    updates:
      - package-ecosystem: "gomod"
        directory: "/"
        schedule:
          interval: "weekly"
        groups:
          kubernetes:
            patterns:
              - "*k8s.io*"
          ...

### [question] site/go.mod left unconfigured
- where: `.github/dependabot.yaml:1-20`
- concern: `site/go.mod` has no overlap with the defined groups currently, so omitting it is low-impact, but it also means site deps get no automated version updates at all (only security alerts). Worth confirming this is intentional rather than an oversight.
- excerpt: |
    (no site/go.mod entry in dependabot.yaml)

### [nit] no open-pull-requests-limit set
- where: `.github/dependabot.yaml:1-20`
- concern: Defaults to 5 concurrent version-update PRs. Probably fine given grouping reduces volume, but since this is the first gomod config in the repo, worth being a deliberate choice rather than an implicit default.
- excerpt: |
    schedule:
      interval: "weekly"

## Checked
- Group glob patterns (`*k8s.io*`, `*github.com/aws*`, `*github.com/prometheus*`, `*golang.org/x*`) are valid dependabot syntax and match the intended package families in root `go.mod` (53 matching lines).
- No existing `.github/dependabot.yaml`/`.yml` prior to this PR (confirmed via git history) — config-only addition, no behavior to regress.
- Change is additive/config-only: no code, no security-sensitive surface, low blast radius if group definitions are imperfect (worst case is suboptimal PR batching).

## Open questions
- Was omitting `hack/tools` from the grouped config intentional, or should a second `gomod` entry (directory: "hack/tools") with matching groups be added to fully close the churn gap seen in #806/#807?
- Any plan to add `site/go.mod` for version updates, or is that module intentionally left to security-alert-only updates?
