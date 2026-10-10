---
pr: kubernetes-sigs/prow#999
title: "Restart PCM when disabled clusters change"
head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
base: main
reviewed_at: 2026-10-09T15:53:10Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-10T16:01:08Z
  gated_head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
  reviewed_head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
---

# Review

## Gate

**Decision: merge**

The PR remains open at the reviewed head; there are no new commits since review. The local review contains no gating findings, and GitHub has no submitted reviews, substantive inline comments, or actionable holds. The independent pass found no API or configuration compatibility changes.

**Gating list:** None.

**Independent merge risk:** `cmd/prow-controller-manager/main.go:133-140,259-269` adds a restart when disabled-cluster membership changes. Existing Kubernetes deployments will restart PCM on this config change; the sample deployment runs one replica, so controller activity can pause briefly. Jobs may still fail during the transition, as the PR description documents. This is the intended behavior, requires no migration, and is not an unacceptable merge risk. No separate release-note file changed; the PR description documents the behavior. It is not opt-in or rollout-gated.

## Verdict

Approve. Code quality, maintainability, and deployment risk reviewers all found no actionable issues. The change uses PCM's existing graceful shutdown path to rebuild clients and watches that are initialized only at startup, and introduces no configuration or API compatibility changes.

## What this PR does

- Subscribes PCM to config updates before initializing cluster clients.
- Compares old and new disabled-cluster membership as sets.
- Requests graceful shutdown when membership changes so the process restarts with rebuilt clients and pod watches.
- Adds focused tests for enabling, disabling, replacing, and equivalent disabled-cluster sets, plus a canceled watcher.

## Findings

No findings.

## Checked

- Code quality: watcher control flow, shutdown handling, channel closure, set comparisons, and test coverage; reviewer verdict was APPROVE.
- Maintainability: scope, reuse of existing config and interrupt mechanisms, logging, and test structure; maintenance burden assessed LOW.
- Deployment risk: no configuration schema, API, or CLI changes; existing configs remain compatible; risk assessed LOW.
- Operators may see a brief PCM interruption on restart. Jobs may still fail during the transition because assignment, config propagation, and shutdown are not coordinated; this limitation is described in the PR.
- Tests were reviewed but not run during this review.

## Open questions

None.
