---
pr: kubernetes-sigs/prow#744
title: "tide: optionally abort superseded batch jobs before retriggering"
head_sha: 48bf3485663fc106802442efda3d2a87e5c91dd3
base: main
reviewed_at: 2026-09-22T22:30:05Z
verdict: needs-discussion
refresh_log:
  - prev_sha: f33597699b8a233b37560b4949c03794dbc5633e
    new_sha: 48bf3485663fc106802442efda3d2a87e5c91dd3
    summary: "Force-push/rebase onto current main. The feature patch remains equivalent; reopened a documentation finding because the documented config literal is false while the stated and effective default is true."
  - prev_sha: 230403b27be3fe1b9ca8c18fef9db3fe627118be
    new_sha: ce86882fae44ed61cd4058cc30f43478200ed31b
    summary: "Rebase onto main (no PR code changes). Discussion: two maintainers favor default-true; author amenable. LGTM removed by rebase."
  - prev_sha: ce86882fae44ed61cd4058cc30f43478200ed31b
    new_sha: f33597699b8a233b37560b4949c03794dbc5633e
    summary: "Author flipped default to true. Config, accessor, test, and doc all updated. CI lint failing."
  - prev_sha: f33597699b8a233b37560b4949c03794dbc5633e
    new_sha: f33597699b8a233b37560b4949c03794dbc5633e
    summary: "No code changes. carterpewpew pinged for status (2026-08-10); still blocked only on lgtm/approved labels per gate."
gate:
  decision: merge
  gated_at: 2026-09-22T22:43:40Z
  gated_head_sha: 48bf3485663fc106802442efda3d2a87e5c91dd3
  reviewed_head_sha: 48bf3485663fc106802442efda3d2a87e5c91dd3
---

## Gate

**Decision: merge** — The rebased feature patch is unchanged since the refreshed review, has targeted test coverage, and has no unresolved substantive reviewer feedback. The generated documented-config placeholder is consistent with the existing `prioritize_existing_batches` convention and does not configure a global value.

### Gating list
- **addressed / no action**: The `"": false` value at `pkg/config/prow-config-documented.yaml:1486-1487` is the generated placeholder convention also used by `prioritize_existing_batches`; lookup only treats `"*"`, an org, or an org/repo key as configuration.
- **acceptable follow-up**: Successful aborts have no per-job info log (`pkg/tide/tide.go:1562-1571`) and no metric. These affect observability only; failures are logged and do not prevent retriggering.

### Merge risk (Area 2)
- **Configuration**: Additive `abort_superseded_batch_jobs` map with `omitempty`; existing configurations retain the default and parse unchanged. No CRD, proto, or migration impact.
- **Behavioral**: Default `true` affects every Tide batch-merging deployment by aborting stale work that Tide can no longer use. Operators can retain prior behavior with `"*": false`; include this intentional default change in release notes.
- **API surface**: The config field and accessor are additive; no exported surface was removed or signature-changed.

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
### [should-fix] Documented config example contradicts the effective default
- where: `pkg/config/prow-config-documented.yaml:1482-1487`
- resolution: Not a defect. This is the generator's placeholder form for `map[string]bool`, matching `prioritize_existing_batches` (`pkg/config/prow-config-documented.yaml:1624-1628`). The accessor recognizes only `"*"`, org, and org/repo keys, so the empty key does not override the true fallback.

### [should-fix] Default should be true, not false
- where: `pkg/config/tide.go:407`
- resolution: Addressed in `f335976`. The accessor, comment, and tests still default to `true` after the rebase; the documented-config portion was reopened as a separate finding because its example is now `"": false`.

### [question] Should this default to true?
- where: `pkg/config/tide.go:252`
- resolution: Promoted to [should-fix] "Default should be true, not false" based on maintainer consensus (petr-muller + stmcginnis). Now addressed in `f335976`.

## Checked
- Config accessor `AbortSupersededBatchJobs()` follows exact same repo>org>global fallback as `PrioritizeExistingBatches()`
- Default true — flipped per maintainer consensus; comment, godoc, and tests match. The documented YAML uses the standard generated empty-key placeholder shared by equivalent map fields.
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

## Activity since 2026-06-20T20:00:00Z
- **carterpewpew** (2026-08-10): Pinged petr-muller and stmcginnis asking whether any more changes are needed. No code changes — PR remains blocked only on the `lgtm`/`approved` labels documented in the gate.

## Activity since 2026-08-11T14:47:08Z
- **carterpewpew** (2026-09-18): Force-pushed/rebased the PR to `48bf3485663fc106802442efda3d2a87e5c91dd3`; the PR now has one feature commit on current `main`.
- **kubernetes-prow[bot]** (2026-09-18): Reported that the PR is not approved and needs an approver for `pkg/OWNERS` (droslean); current labels remain `cncf-cla: yes`, `ok-to-test`, `size/L`, and `area/tide`.
