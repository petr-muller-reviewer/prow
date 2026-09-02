---
pr: 691
title: "Document sinker component"
head_sha: "8738db108e8cd0bcfc808328fd23ef7078f879f2"
base: main
reviewed_at: "2026-06-01T00:31:09Z"
verdict: APPROVE_WITH_SUGGESTIONS
refresh_log:
  - old_sha: ""
    new_sha: "8738db108e8cd0bcfc808328fd23ef7078f879f2"
    refreshed_at: "2026-06-01T00:31:09Z"
    summary: "No code changes; /ok-to-test granted by ameukam on 2026-05-23"
gate:
  decision: merge
  gated_at: "2026-09-02T15:58:17Z"
  gated_head_sha: "8738db108e8cd0bcfc808328fd23ef7078f879f2"
  reviewed_head_sha: "8738db108e8cd0bcfc808328fd23ef7078f879f2"
---

# PR #691: Document sinker component

**Author:** [VPC-byte](https://github.com/VPC-byte) (first-time contributor)
**Date:** 2026-04-27
**Diff:** +62 / -2
**Labels:** `area/documentation` `size/M`
**Netlify Preview:** https://deploy-preview-691--k8s-prow.netlify.app

## Verdict

> **APPROVE WITH MINOR SUGGESTIONS**

## Gate

**Decision: merge.** PR #691 is now merged at `8738db108e8cd0bcfc808328fd23ef7078f879f2`, the same commit the review covered. The change is documentation-only (`site/content/en/docs/components/core/sinker.md`, +62/-2); it introduces no API, configuration-schema, flag, or runtime behavior change.

### Gating list

- None. There are no blocking or should-fix findings, reviewer change requests, substantive inline comments, or explicit holds.
- The review observations on orphaned-pod cleanup, ProwJob verbs, `max_pod_age` semantics, and metrics organization remain useful follow-up improvements, but do not create deployment or compatibility risk.

### Independent merge risk

- No notable merge risk. Existing deployments are unaffected because the PR changes only a rendered documentation page.

## Since Previous Review (refreshed 2026-06-01)

- No code changes — single commit `8738db10` remains the PR head.
- 2026-05-23: **ameukam** granted `/ok-to-test` — CI tests will now run on this PR.
- 2026-04-27: **stmcginnis** commented `/close #622` — not a valid prow command (prow only recognizes bare `/close`), no effect.
- No inline review comments or formal reviews submitted. PR still needs `/lgtm` and approval from **krzyzacy** (as designated approver).

## Summary

Replaces the empty placeholder page for the Sinker component (`site/content/en/docs/components/core/sinker.md`) with documentation covering cleanup behavior, configuration, required RBAC, CLI flags, and metrics. Fixes #495. All claims were verified against source code in `cmd/sinker/main.go` and `pkg/config/config.go`.

## Verification Matrix

| Claim | Source | Status |
|-------|--------|--------|
| Config fields: `resync_period`, `max_prowjob_age`, `max_pod_age`, `terminated_pod_ttl`, `exclude_clusters` | `pkg/config/config.go:1098-1112` | Accurate |
| Defaults: 1h resync, 168h ProwJob age, 24h pod age, TTL = pod age | `pkg/config/config.go:2683-2696` | Accurate |
| Cleanup: aged ProwJobs, aged periodics (retaining latest), aged pods, TTL'd pods | `cmd/sinker/main.go:334-484` | **Incomplete** — orphaned pods not mentioned |
| `max_pod_age` description: "older than max_pod_age" | `cmd/sinker/main.go:458-460` | **Imprecise** — measured from start time, only when ProwJob complete |
| CLI flags: `--config-path`, `--job-config-path`, `--kubeconfig`, `--dry-run` (default true), `--run-once` | `cmd/sinker/main.go:76-87` | Accurate |
| Metrics under `sinker_` prefix | `cmd/sinker/main.go:263-316` | **Mostly** — `job_configmap_size` lacks prefix |
| ProwJob RBAC: "list and delete" | `test/integration/.../sinker_rbac.yaml:17-21` | **Incomplete** — also needs `watch`, `get` |
| Pod RBAC: "list, watch, get, patch, delete" | `test/integration/.../sinker_rbac.yaml:64-69` | Accurate |

## Converging Concerns

*Issues flagged independently by 2+ reviewers carry the highest confidence.*

### [Converging — Maintainability + Deployment Risk] Orphaned pod cleanup is not documented

Sinker also deletes pods whose ProwJob no longer exists (`reasonPodOrphaned` / `isPodOrphaned` in source). This is operationally relevant behavior distinct from the age/TTL rules and should be mentioned under "What Sinker Cleans Up."

### [Converging — Maintainability + Deployment Risk] `job_configmap_size` metric lacks the `sinker_` prefix

The doc claims all metrics are under the `sinker_` prefix, but this metric doesn't follow that convention. Operators searching by prefix would miss it.

## Additional Findings

### [Suggestion — line 46] ProwJob access verbs are understated

The doc says sinker needs to "list and delete" ProwJobs but the RBAC manifest and code show `list`, `watch`, `get`, and `delete` are all required. The pod verbs on line 48 are already listed in full, so ProwJob verbs should be consistent.

```diff
-Sinker needs access to the infrastructure cluster to list and delete ProwJobs in
+Sinker needs access to the infrastructure cluster to list, watch, get, and delete ProwJobs in
```

### [Suggestion — line 20] `max_pod_age` semantics are imprecise

The doc says "Test pods created by Prow that are older than `max_pod_age`" but the age is actually measured from **pod start time** (not creation time), and only applies when the associated ProwJob is complete. A long-running pod whose ProwJob is still active will not be reaped, which could surprise operators tuning this value.

### [Nit — lines 66-68] Metrics are buried in the CLI Flags section

The last paragraph about Prometheus metrics is appended to "CLI Flags" rather than having its own `## Metrics` heading. Since the documentation otherwise has clean section boundaries, a dedicated heading would be more discoverable and consistent.

## What's Good

- Clean replacement of empty placeholder — no leftover boilerplate
- Config example shows all fields with realistic default values
- Correctly explains periodic ProwJob retention for horologium continuity
- Notes the `--dry-run=true` default and what operators must do for production
- Uses backtick-formatted names matching actual YAML config keys
- Prose is concise and operator-focused — no unnecessary padding

## Ready-to-Post Review Comments

### Comment 1: Orphaned pod cleanup (converging concern)

```
suggestion: Sinker also cleans up orphaned pods — pods whose associated ProwJob no longer exists (see `reasonPodOrphaned` / `isPodOrphaned` in `cmd/sinker/main.go`). This is a distinct cleanup behavior from the age/TTL rules and worth mentioning. Consider adding a bullet here:

```suggestion
* Pods whose associated ProwJob no longer exists (orphaned pods).
```
```

### Comment 2: ProwJob verbs

```
suggestion: The RBAC manifest (`test/integration/config/prow/cluster/sinker_rbac.yaml`) and the controller-runtime manager setup show that sinker also needs `watch` and `get` on ProwJobs, not just `list` and `delete`. Since you already list all five verbs for pods on line 48, it'd be consistent to do the same here:

```suggestion
Sinker needs access to the infrastructure cluster to list, watch, get, and delete ProwJobs in
```
```

### Comment 3: max_pod_age semantics

```
nit: Worth noting that `max_pod_age` is measured from the pod's start time (not creation time), and only applies to pods whose ProwJob has completed. A still-running pod won't be reaped even if it exceeds this age. This nuance matters for operators tuning the value.
```

### Comment 4: Metrics section

```
nit: The metrics paragraph at the end feels like it belongs under its own `## Metrics` heading rather than being tacked onto the CLI Flags section. Optional, but would make the page easier to scan.
```

## Advisor Synthesis

All three independent reviewers unanimously approve. The change carries zero deployment risk and significantly reduces documentation debt. Two reviewers independently flagged orphaned pod cleanup omission and the `job_configmap_size` prefix inconsistency. These are worth correcting but should not block the merge.

### Ready-to-post PR comment (full review)

```
This is a solid documentation contribution that replaces a placeholder with accurate, well-structured content. I am approving this as-is. A few non-blocking suggestions for a follow-up commit:

1. Sinker also cleans up orphaned pods (pods whose ProwJob no longer exists) — worth adding a bullet under "What Sinker Cleans Up"
2. The ProwJob RBAC section understates the required verbs — sinker also needs `watch` and `get`, not just `list` and `delete`
3. `max_pod_age` is measured from pod start time and only applies when the ProwJob is complete — could be clarified
4. The metrics paragraph would be more discoverable under its own `## Metrics` heading

None of these block the merge. Thanks for the contribution!

/ok-to-test
/lgtm
```

## Post-merge follow-up

After PR #691 merges, carry over the strongest additions from closed PR #622:

- Document orphaned-pod cleanup (pods whose associated ProwJob no longer exists).
- Add explicit defaults for each `sinker` configuration field and a link to `cmd/sinker`.
- Expand the flag reference where useful, while retaining a focused operator-facing page.
- Preserve the `max_pod_age` qualification: it is based on pod start time and does not delete pods whose associated ProwJob is still active.

## Reviewer Actions

### Before approving

- First-time contributor — will need `/ok-to-test`
- Netlify preview deployed and renderable
- Ask for ProwJob verb fix (or accept as-is)

### Prow commands

```
/ok-to-test
/lgtm
```
