---
pr: kubernetes-sigs/prow#744
title: "tide: optionally abort superseded batch jobs before retriggering"
head_sha: f33597699b8a233b37560b4949c03794dbc5633e
base: main
reviewed_at: 2026-06-22T17:31:16Z
verdict: approve
refresh_log:
  - prev_sha: 230403b27be3fe1b9ca8c18fef9db3fe627118be
    new_sha: ce86882fae44ed61cd4058cc30f43478200ed31b
    summary: "Rebase onto main (no PR code changes). Discussion: two maintainers favor default-true; author amenable. LGTM removed by rebase."
  - prev_sha: ce86882fae44ed61cd4058cc30f43478200ed31b
    new_sha: f33597699b8a233b37560b4949c03794dbc5633e
    summary: "Author flipped default to true. Config, accessor, test, and doc all updated. CI lint failing."
gate:
  decision: hold
  gated_at: 2026-06-22T18:03:11Z
  gated_head_sha: f33597699b8a233b37560b4949c03794dbc5633e
  reviewed_head_sha: f33597699b8a233b37560b4949c03794dbc5633e
---

## Gate

**Decision: hold** — Code is correct and all review findings are addressed or acceptably dispositioned. The only blockers are process: missing `lgtm` and `approved` labels. Once stmcginnis re-LGTMs (he LGTM'd the original, it was removed by rebase + the default-flip push) and a maintainer `/approve`s, this is clear to merge.

### Gating list
- **addressed**: Default flipped to true (`pkg/config/tide.go:407`). Done in `f335976`.
- **not-addressed, non-blocking**: No logging on successful abort (`pkg/tide/tide.go:1561-1571`). Acceptable to merge as-is; can be a fast follow-up.
- **not-addressed, non-blocking**: No Prometheus counter (`pkg/tide/tide.go:1541-1573`). Acceptable as follow-up PR.
- **label**: `lgtm` missing. stmcginnis LGTM'd on 2026-06-08; label was removed by subsequent pushes. Needs re-application.
- **label**: `approved` missing. No maintainer has `/approve`d yet.

### Merge risk (Area 2)
- **Configuration**: New additive field `abort_superseded_batch_jobs` with `omitempty`. Existing configs parse without issue. No breaking change.
- **Behavioral**: Default is `true`. All existing Tide installations will automatically abort superseded batch jobs on upgrade. The aborted jobs were already producing unusable results (invisible to Tide after baseSHA advance), so the impact is net positive. Operators who want the old behavior can set `"*": false`. Should be release-noted.
- **API surface**: No exported Go API changes beyond the new config field and accessor, both additive. No CRD/proto changes.
- **Blast radius**: Every Prow installation that uses Tide batch merging. Mitigated by the fact that the aborted work was already wasted, and by the per-org/repo opt-out mechanism.

## Findings

### [should-fix] No logging on successful abort
- where: `pkg/tide/tide.go:1561-1571`
- concern: The patch loop logs errors but is silent on success. An info-level log per aborted job (name + old baseSHA) would match the codebase's instrumentation standard and aid production debugging. Flagged independently by all three review perspectives.
- excerpt: |
    if err := c.prowJobClient.Patch(c.ctx, &pj, ctrlruntimeclient.MergeFrom(prev)); err != nil {
        errs = append(errs, fmt.Errorf("failed to abort superseded batch prowjob %s: %w", pj.Name, err))
    }

### [should-fix] No Prometheus counter for aborted jobs
- where: `pkg/tide/tide.go:1541-1573`
- concern: The codebase already has `tideMetrics.retests` for batch triggers. An analogous `tide_superseded_batch_aborts_total` counter (with org/repo/branch labels) would let operators quantify compute savings and detect misconfiguration. Could be a follow-up PR.

### [nit] List-to-Patch race can mark a just-completed job as aborted
- where: `pkg/tide/tide.go:1562`
- concern: Between the List and the Patch loop, a job could complete naturally. The `pj.Complete()` guard only checks the snapshot from list time. The window is narrow and the consequence is cosmetic (the work already ran), but worth documenting as a known limitation.
- excerpt: |
    if pj.Spec.Refs == nil || pj.Spec.Refs.BaseSHA == sp.sha || pj.Complete() {
        continue
    }

### [nit] Test uses branch name as BaseSHA for "current" job
- where: `pkg/tide/tide_test.go:2110`
- concern: The "current-batch" fixture sets `BaseSHA: defaultBranch` (a branch name like "master"), which happens to equal `sp.sha`. Using a distinct fake value like `"current-sha"` with a matching `sp.sha` would make the test intent clearer.

### [question] Interaction with prioritize_existing_batches
- where: `pkg/tide/tide.go:1602-1607`
- concern: Both features affect batch lifecycle. `prioritize_existing_batches` reuses existing batch results; `abort_superseded_batch_jobs` cleans up when a new batch is triggered. They are complementary, but the interaction may not be obvious to operators. Worth a brief note in config documentation?

## Resolved
### [should-fix] Default should be true, not false
- where: `pkg/config/tide.go:407`
- resolution: Addressed in `f335976`. Default flipped from `false` to `true`. Config comment, accessor godoc, test case, and `prow-config-documented.yaml` all updated consistently.

### [question] Should this default to true?
- where: `pkg/config/tide.go:252`
- resolution: Promoted to [should-fix] "Default should be true, not false" based on maintainer consensus (petr-muller + stmcginnis). Now addressed in `f335976`.

## Checked
- Config accessor `AbortSupersededBatchJobs()` follows exact same repo>org>global fallback as `PrioritizeExistingBatches()`
- Default true — flipped per maintainer consensus; comment, godoc, test, and documented-yaml all updated
- Config test covers all resolution levels (unset, global, org, repo override, unmatched-org fallback)
- `DeepCopy` + `MergeFrom` patch pattern is correct for controller-runtime
- `utilerrors.NewAggregate` for error collection matches codebase patterns
- Abort is non-fatal: logged error, batch triggering proceeds
- Placement in `takeAction` correct: only runs when `batchPending == 0`
- Label selector on list query matches labels set in `trigger()` (CreatedByTideLabel, ProwJobTypeLabel, OrgLabel, RepoLabel, BaseRefLabel)
- Test exercises both stale-SHA job (aborted) and current-SHA job (skipped)
- Whitespace changes in `tide_test.go` are gofmt artifacts from new longer field names
- Pre-existing job filtering refactored from `reflect.DeepEqual` to name-based `sets.Set` — cleaner and necessary since aborted jobs no longer match their original object
- No new RBAC requirements, no config migration needed, no upgrade ordering dependency
- Rollback safe: removing config key or setting `"*": false` returns to previous behavior

## Activity since 2026-06-08T17:25:04Z
- **petr-muller** (2026-06-08): Asked whether default should be true instead of false.
- **carterpewpew** (2026-06-11): Author replied — chose opt-in for safety, willing to flip default if maintainers prefer.
- **k8s-ci-robot** (2026-06-11): LGTM label removed (triggered by rebase push).
- **stmcginnis** (2026-06-11): Agrees default-on makes sense — superseded batches produce unusable results.
- **petr-muller** (2026-06-16): `/ok-to-test`.
- **carterpewpew** (2026-06-20): Flipped default to true as suggested. Pushed `f335976`.
- **kubernetes-prow[bot]** (2026-06-20): `pull-prow-verify-lint` failing on `f335976`.

## Open questions
- Would you consider adding an info-level log per successfully aborted job? Something like `sp.log.WithField("prowjob", pj.Name).Info("Aborted superseded batch job")` — low effort, high operational value.
- Is the List-to-Patch race on job completion worth documenting in a code comment, or is it acceptable given the narrow window and cosmetic consequence?
- Would a brief note in the config documentation about the interaction with `prioritize_existing_batches` be useful for operators?
