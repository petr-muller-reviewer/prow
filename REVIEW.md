---
pr: kubernetes-sigs/prow#679
title: "Add Tekton preset support for PipelineRun jobs"
head_sha: 0c8fa0386c71b5b4bec7de564f648c99bc1e7589
base: main
reviewed_at: 2026-07-26T23:44:21Z
verdict: request-changes
---

## What this PR does

- Extends the existing `Preset` mechanism (currently PodSpec-only) with Tekton `PipelineRunSpec` fields: `tekton_params`, `tekton_workspaces`, `tekton_service_account`, `tekton_timeout`, `tekton_task_run_template`.
- Adds `mergeTektonPreset`/`ResolveTektonPresets` (`pkg/config/jobs.go`, `pkg/config/config.go`) mirroring the existing `mergePreset`/`resolvePresets` PodSpec logic, with label matching and duplicate-field conflict detection.
- Wires resolution into `defaultPresubmits`/`defaultPostsubmits`/`DefaultPeriodic` for jobs that `HasPipelineRunSpec()`.
- Adds deepcopy support for the new `Preset` fields (`zz_generated.deepcopy.go`).
- Adds unit tests (`config_test.go`, `jobs_test_tekton_presets.go`) and an integration test (`pipeline_preset_test.go`) verifying end-to-end propagation into created `PipelineRun`s.

## Findings

### [blocking] Unit test file is misnamed and never runs
- where: `pkg/config/jobs_test_tekton_presets.go`
- concern: File declares `package config`, imports `"testing"`, and defines `TestResolveTektonPresets`/`TestMergeTektonPreset` (287 lines, 15 test cases), but the filename ends in `_presets.go`, not `_test.go`. Go's toolchain only excludes files ending exactly in `_test.go` from normal builds, so this file compiles into every production binary that imports `pkg/config` (hook, deck, sinker, prow-controller-manager, etc.), permanently pulling `testing` into production, AND `go test ./pkg/config/...` never discovers or runs these tests. CI reports green while this dedicated unit-test coverage for the merge logic silently never executes — confirmed independently by three separate review passes. Fix: rename to `jobs_tekton_presets_test.go` (or fold into `jobs_test.go`).
- excerpt: |
    // pkg/config/jobs_test_tekton_presets.go — WRONG suffix, should be *_test.go
    package config

    import (
        "testing"
        pipelinev1 "github.com/tektoncd/pipeline/pkg/apis/pipeline/v1"
        ...
    )

    func TestResolveTektonPresets(t *testing.T) { ... }
    func TestMergeTektonPreset(t *testing.T) { ... }

### [blocking] PR diff is dominated by unrelated/stale churn
- where: `test/integration/config/prow/cluster/50_crd.yaml` (+9303/-52053), plus scattered unrelated diffs in `pkg/config/jobs.go`, `pkg/config/config.go`, `test/integration/test/deck_test.go`, `test/integration/test/sinker_test.go`, `pkg/entrypoint/run_test.go`
- concern: The CRD file diff is purely a `controller-gen` version bump (`v0.6.3-0.20210827222652-7b3a8699fa04` → `v0.17.3`), unrelated to Tekton presets. Additional unrelated changes appear: `slices.Contains`/`slices.ContainsFunc`/`maps.Copy` modernizations, `Job`/`Type` field additions to unrelated `ProwJobSpec` fixtures, dropped `tt := tt` idioms, and an unrelated shell-snippet change in `run_test.go`. Indicates the branch is stale relative to `main` and needs a rebase — likely to merge-conflict on the generated CRD file as-is, and makes the actual feature diff hard to review/bisect independently.
- excerpt: |
    -    controller-gen.kubebuilder.io/version: v0.6.3-0.20210827222652-7b3a8699fa04
    -  creationTimestamp: null
    +    controller-gen.kubebuilder.io/version: v0.17.3

### [should-fix] Timeout conflict check misses explicit zero-duration timeouts
- where: `pkg/config/jobs.go:151-159` (`mergeTektonPreset`)
- concern: Conflict is only flagged when `Timeouts.Pipeline != nil && Duration != 0`. A job that explicitly sets `Duration: 0` (meaning "no timeout") would have a preset's timeout silently applied instead of erroring, unlike every other field in this function (e.g. service account, which correctly distinguishes "unset" via empty string vs. any explicit value).
- excerpt: |
    if pipelineRunSpec.Timeouts.Pipeline != nil && pipelineRunSpec.Timeouts.Pipeline.Duration != 0 {
        return fmt.Errorf("timeout conflict: both preset and spec define timeout")
    }
    pipelineRunSpec.Timeouts.Pipeline = preset.TektonTimeout.DeepCopy()

### [should-fix] Discarded error from GetPipelineRunSpec(), triplicated call site
- where: `pkg/config/config.go:2108-2114, 2133-2139, 2153-2160`
- concern: `pipelineRunSpec, _ := presubmits[idx].GetPipelineRunSpec()` discards the error at three near-identical call sites (presubmits, postsubmits, periodics). Currently safe because each is gated by a preceding `HasPipelineRunSpec()` check with matching logic, but the invariant is implicit — if the two methods ever diverge, this becomes a silent no-op (nil spec passed through) rather than a surfaced bug. The triplication also means any future fix here needs three coordinated edits; a shared helper (e.g. `resolveTektonPresetsForJob(jb JobBase, ...)`) would remove both problems at once.
- excerpt: |
    if presubmits[idx].HasPipelineRunSpec() {
        pipelineRunSpec, _ := presubmits[idx].GetPipelineRunSpec()
        if err := ResolveTektonPresets(ps.Name, ps.Labels, pipelineRunSpec, append(c.Presets, additionalPresets...)); err != nil {

### [nit] mergeTektonPreset conflict semantics undocumented and asymmetric
- where: `pkg/config/jobs.go:112-170` (`mergeTektonPreset`)
- concern: Service-account/timeout/pod-template use "skip if already equal, error only if different" merge semantics, while params/workspaces always hard-error on any duplicate (even identical values) — mirroring `mergePreset`'s pod-spec behavior for slices but diverging for scalars. Nothing comments on why this split exists, so a future contributor adding a new Tekton preset field has no documented rule to follow.

### [nit] ResolveTektonPresets exported without an accurate reason given
- where: `pkg/config/config.go:2980-2982`
- concern: Doc comment says it's "exported to allow testing," but the in-package tests could call `mergeTektonPreset` directly (as `TestMergeTektonPreset` already does). The actual reason for exporting is that `test/integration/test/pipeline_preset_test.go` calls `config.ResolveTektonPresets` from a different package — the comment should say that instead.
- excerpt: |
    // ResolveTektonPresets resolves and applies Tekton presets to a PipelineRunSpec based on label matching.
    // This function is exported to allow testing of preset application logic.
    func ResolveTektonPresets(name string, labels map[string]string, pipelineRunSpec *pipelinev1.PipelineRunSpec, presets []Preset) error {

### [nit] append(c.Presets, additionalPresets...) computed twice per job
- where: `pkg/config/config.go:2105-2111, 2130-2136`
- concern: In the presubmit/postsubmit defaulting loops, `append(c.Presets, additionalPresets...)` is now evaluated once for `resolvePresets` and again for `ResolveTektonPresets`, each allocating a new backing slice with identical contents. Compute once per iteration and reuse.

### [question] TektonTaskRunTemplate.PodTemplate merge path untested
- where: `pkg/config/jobs.go:162-167`
- concern: Neither `config_test.go`'s `TestTektonPresetIntegration` nor `jobs_test_tekton_presets.go`'s table tests exercise the `PodTemplate` merge/conflict branch — only service account, params, workspaces, and timeout are covered.

## Checked
- `mergeTektonPreset` label-matching, duplicate-param/workspace/service-account conflict detection — correct and mirrors `mergePreset` pattern.
- Deepcopy additions for new `Preset` fields — correctly generated, covers `TektonParams`, `TektonWorkspaces`, `TektonTimeout`, `TektonTaskRunTemplate`.
- Unit test coverage design (label match/mismatch, duplicate detection, multi-preset application, default label-less preset) is good in principle — undermined by the file-naming bug above.
- Integration test (`pipeline_preset_test.go`, correctly named `*_test.go`): verifies preset fields propagate into created `PipelineRun` objects, including merging preset params with job-specific params.
- Config compatibility: all new `Preset` fields are `omitempty` additions; existing presets.yaml files parse and behave unchanged; new merge path only fires for jobs that already opt into `PipelineRunSpec`.
- No security concerns — merges already-trusted Prow config data into pipeline specs, same trust boundary as existing PodSpec presets.

## Open questions
- Can the misnamed test file be renamed to `jobs_tekton_presets_test.go` and confirmed to run under `go test ./pkg/config/...` before merge?
- Can this branch be rebased onto current `main` to drop the unrelated CRD regen and modernization/test-fixture changes?
- Is the zero-duration timeout case (explicit "no timeout") intentionally excluded from conflict detection, or an oversight?
- Is the scalar-vs-slice conflict-semantics split in `mergeTektonPreset` intentional? Worth a comment either way.
