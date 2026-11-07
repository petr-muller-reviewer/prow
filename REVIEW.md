---
pr: kubernetes-sigs/prow#1058
title: "Add affinity rules to presets and allow presets to be overriden in jobs"
head_sha: e5d7addebd65806882640b7ff98cec9289fd4d05
base: main
reviewed_at: 2026-11-07T21:58:25Z
verdict: approve
---

# Review

## Verdict

Approve with suggestions: the merge is correct, tested, and documented; the follow-ups are about making affinity and diagnostic behavior explicit, not fixing bugs.

All three maintainer perspectives (code quality, maintainability, deployment risk) approved the change: no correctness bugs, no forward-breaking config changes, and no CRD/RBAC/API changes. The most significant follow-up is that preset affinity now populates `PodSpec.Affinity` and thereby suppresses a configured `decoration_config.scheduling_options.affinity` — a pre-existing interaction that this feature makes much easier to hit. The rest are affinity-term deduplication, diagnostics, an unstated one-shot contract, and small test gaps.

## What this PR does

- Adds `affinity` to `Preset` and merges it into the pod spec: preferred node/pod/pod-anti-affinity terms and required pod/pod-anti-affinity terms are appended; required node affinity may be set only once (a second preset setting it is an error).
- Lets a job override a preset's env var, volume, volume mount (matched by name) and required node affinity; the preset's value is skipped and an Info message is logged.
- Treats an identical toleration on the job as deduplicated, appends non-identical preset tolerations, and still errors on duplicate tolerations between presets.
- Adds `job`/`preset` log fields and downgrades the override message from Warn to Info.
- Documents the new field, the affinity merge rules, and the job-over-preset precedence.

## Findings

### [should-fix] Affinity terms are appended without deduplication
- where: `pkg/config/jobs.go:153-198`
- concern: Preferred node terms and required pod/pod-anti-affinity terms are appended unconditionally. An identical term on the job is duplicated (a duplicated preferred term doubles its scheduling weight), and a diamond include graph (two presets including the same base) silently duplicates terms while env/volume/mount collisions on the same path error. Dedupe with `cmp.Equal` as tolerations do, or document append-only semantics explicitly, and add a diamond-include test.
- excerpt: |
    dst.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution = append(
        dst.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution,
        src.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution...)

### [should-fix] Preset affinity suppresses scheduling_options.affinity
- where: `pkg/pod-utils/decorate/podspec.go:846-852`
- concern: Decoration applies `scheduling_options.affinity` only when `spec.Affinity == nil`. A preset that adds even one preferred term makes `spec.Affinity` non-nil, so a cluster/repo-level required affinity default is silently dropped and jobs can land outside the intended node pool. This is a pre-existing interaction (tolerations already behave this way) but preset affinity makes it much easier to hit; merge the default or warn when it is suppressed.
- excerpt: |
    if pj.Spec.DecorationConfig.SchedulingOptions.Affinity != nil && spec.Affinity == nil {
        spec.Affinity = pj.Spec.DecorationConfig.SchedulingOptions.Affinity
    }

### [should-fix] Override and conflict diagnostics are not actionable
- where: `pkg/config/jobs.go:133-141`
- concern: The remaining failure path — two presets colliding on an env var/volume/mount — reports only the key, not the presets involved, so diagnosing it means replaying label matching by hand. Overrides log once per container per value with no container identity, and every anonymous preset renders as `preset=<none>`. Capture preset identity (and container index) in errors, and log once per preset+name.
- excerpt: |
    if !slices.ContainsFunc(job, match) {
        return false, fmt.Errorf("%s duplicated in pod spec: %s", kind, name)
    }
    log.Infof("Job overrides %s %s set by preset.", kind, name)

### [should-fix] The one-shot resolvePresets contract is unstated
- where: `pkg/config/config.go:3029`, `pkg/config/jobs.go:77`
- concern: Correctness depends on `job` being a pristine snapshot taken exactly once per spec: a second `resolvePresets` call would treat preset-applied values as job values (turning preset-vs-preset errors into silent skips) and re-append preferred terms. Every current path resolves once (the in-repo cache deep-copies before defaulting), so there is no live bug, but the invariant lives only in reviewers' heads. Document it next to the snapshot.
- excerpt: |
    job := spec.DeepCopy()
    log := logrus.WithField("job", name)

### [nit] Positional job.Containers[i] lookup relies on an undocumented invariant
- where: `pkg/config/jobs.go:82,102`
- concern: The override check indexes `job.Containers[i]` using an index into `podSpec.Containers`. Safe today because the snapshot is of the same spec and presets never add containers, but a future container-bearing preset field would panic rather than error. Add a comment or a length guard.
- excerpt: |
    checkPresetOverride(containers[i].Env, job.Containers[i].Env, match, "env var", e1.Name, log)

### [nit] Doc comments drift from behavior
- where: `pkg/config/jobs.go:57-60,75-77`
- concern: `applyPreset` says "conflicts between presets are errors", which is untrue for affinity (merged by design), and the `Preset.Affinity` comment omits job precedence — only the site docs explain it. Tighten both so the code is self-describing.
- excerpt: |
    // Values set on the job itself (job is the spec before any preset was applied) take
    // precedence over the preset; conflicts between presets are errors.

### [nit] Missing tests for PodAffinity append and repeated required node affinity
- where: `pkg/config/jobs_test.go:809`
- concern: `TestMergePresetAffinity` covers the `PodAffinity == nil` assignment but not the append branch when both job and preset set `PodAffinity`; it also does not cover job-required node affinity combined with two presets that also set required (must log and not error). Both pin the intended precedence.
- excerpt: |
    if src.PodAffinity != nil {
        if dst.PodAffinity == nil {
            dst.PodAffinity = src.PodAffinity

### [nit] Unconditional pod-spec deep copy per job
- where: `pkg/config/config.go:3029`
- concern: `spec.DeepCopy()` runs for every job even when no preset matches, on both the config-load and per-in-repo-config paths. Taking the snapshot lazily once a matching preset is found avoids the copy in the common no-preset case.
- excerpt: |
    job := spec.DeepCopy()

### [nit] Preset conflicts block the whole config reload
- where: `pkg/config/config.go:2222-2237`
- concern: `finalizeJobConfig` returns on the first repo's preset error, so one bad job blocks all config updates for every component (components keep their last-known-good config). The error names the job but not the repo. Document this blast radius, or report preset conflicts as per-job validation errors.
- excerpt: |
    for repo, jobs := range c.PresubmitsStatic {
        if err := defaultPresubmits(jobs, nil, c, repo); err != nil {
            return err
        }

### [question] Volume mounts are matched by name only
- where: `pkg/config/jobs.go:102`
- concern: Matching on `VolumeMount.Name` alone means a job that mounts the same volume at a second path (legal in Kubernetes) silently suppresses the preset's mount. Is that intended, or should the match include `MountPath` / document that any job mount of that volume wins?
- excerpt: |
    match := func(vm2 v1.VolumeMount) bool { return vm1.Name == vm2.Name }

### [question] Additive-only pod affinity has no job opt-out
- where: `pkg/config/jobs.go:168-196`
- concern: Required pod/pod-anti-affinity and preferred terms are only ever appended, so a preset can make a job permanently unschedulable and a job cannot override them (unlike env/volumes/required node affinity). Intended limitation, or should job-set pod affinity replace the preset's?
- excerpt: |
    dst.PodAntiAffinity.RequiredDuringSchedulingIgnoredDuringExecution = append(
        dst.PodAntiAffinity.RequiredDuringSchedulingIgnoredDuringExecution,
        src.PodAntiAffinity.RequiredDuringSchedulingIgnoredDuringExecution...)

## Checked

- Code quality: no correctness bugs. Merge semantics match Kubernetes (required node terms OR-ed, required pod terms AND-ed); `job.Containers[i]` cannot run out of range today; presets are not mutated.
- `go test ./pkg/config/... ./cmd/checkconfig/...`, `go vet ./pkg/config/`, `gofmt -l`, and `git diff --check` are clean.
- `mergeAffinity` deep-copies the preset affinity first, and `TestMergePresetAffinity` diffs the input presets after resolution.
- The `Affinity` deepcopy addition matches the generator's output shape for a `*v1.Affinity` field.
- `resolvePresets`' `spec == nil` early return is equivalent to the previous per-preset nil check.
- Deployment: no forward-breaking config changes, no CRD/RBAC/API changes. Pre-existing job/preset collisions were already errors, so no valid config relied on the old behavior.
- Version skew: `Load` is non-strict, so an older binary silently ignores `presets[].affinity` and runs jobs unplaced; upgrade binaries before adding the field. `checkconfig --unknown-fields-all` uses `LoadStrict` and would flag it.
- Rollback: revert configs that rely on job-overrides before downgrading; the old binary rejects those collisions at load.
- Earlier review feedback (value assertions, preset in the log field, docs, toleration semantics) is addressed in e5d7addeb.

## Open questions

- Should preset affinity merge with, rather than suppress, `decoration_config.scheduling_options.affinity`? A warning is the minimum.
- If a required-node-affinity preset is meant to enforce placement, should there be any way to make it authoritative instead of job-wins?

## Followups

Recorded from `/review:followup` on 2026-11-07. PR merged as `9f4d1868e46d0e0d4588bab8192813699544d859`. Tally: 5 accepted (4 deliver, 1 probe), 4 skipped.

### [correctness] Deduplicate identical affinity terms
- where: `pkg/config/jobs.go:153-198`
- necessity: should
- mode: deliver

```text
In `kubernetes-sigs/prow`, following PR #1058 "Add affinity rules to presets and allow presets to be overriden in jobs" (merged as 9f4d1868e46d0e0d4588bab8192813699544d859), make the preset affinity merge idempotent by deduplicating identical terms.

What and where:
- `pkg/config/jobs.go`, `mergeAffinity` (the node/pod/pod-anti-affinity append blocks around lines 145-200).
- Tests live in `pkg/config/jobs_test.go`, `TestMergePresetAffinity`.

Task:
1. Before appending a preferred node term, a required/preferred pod-affinity term, or a required/preferred pod-anti-affinity term, skip it if an equal term is already present. Use `cmp.Equal` (the toleration path in `applyPreset` already does this) rather than adding a new dependency.
2. Keep the existing required-node-affinity behavior: job wins; two presets setting it is still an error.
3. Add table cases to `TestMergePresetAffinity` for:
   - a job and preset with an identical preferred node term (result contains it once);
   - a diamond include graph where two matching presets include the same base preset carrying an affinity term (result contains the base term once);
   - identical pod-affinity / pod-anti-affinity terms from job and preset.
4. Run `go test ./pkg/config/ -run TestMergePreset -count=1` and `gofmt -l pkg/config`.

Acceptance criteria:
- `mergeAffinity` never appends a term already present in the destination affinity.
- The new table cases fail before the change and pass after it.
- `go test ./pkg/config/` and `gofmt -l pkg/config` are clean.

Scope guard:
- Do not change env/volume/volume-mount/toleration behavior or the required-node-affinity error path.
- Do not add new config fields or change the public `Preset` API.
```

### [deployment] Merge or warn on suppressed scheduling_options.affinity
- where: `pkg/pod-utils/decorate/podspec.go:846-852`
- necessity: should
- mode: deliver

```text
In `kubernetes-sigs/prow`, following PR #1058 "Add affinity rules to presets and allow presets to be overriden in jobs" (merged as 9f4d1868e46d0e0d4588bab8192813699544d859), fix the interaction where a preset-provided `PodSpec.Affinity` silently suppresses `decoration_config.scheduling_options.affinity`.

What and where:
- `pkg/pod-utils/decorate/podspec.go`, function `decorate` (~lines 846-852), where the decoration default is applied only when `spec.Affinity == nil`.
- Preset affinity is merged into the spec at config load in `pkg/config/jobs.go` (`mergeAffinity`).
- Decoration tests live in `pkg/pod-utils/decorate/podspec_test.go`.

Task:
1. Decide and implement merge-or-warn. Preferred: merge the configured default with an existing affinity — append preferred node/pod terms and required pod/pod-anti-affinity terms; for required node affinity present in both, keep the existing (job/preset) value and log a warning rather than silently dropping the configured default.
2. If a full merge is too invasive, implement the minimal safe behavior: log a warning (with job name and the fact that `scheduling_options.affinity` was ignored) whenever `spec.Affinity` is non-nil and `scheduling_options.affinity` is also set.
3. Add a decorate-level unit test covering job/preset affinity plus a configured `scheduling_options.affinity`.
4. Run `go test ./pkg/pod-utils/decorate/ ./pkg/config/`.

Acceptance criteria:
- No code path silently discards a configured `scheduling_options.affinity` without either merging it or logging a warning.
- The new test fails before the change and passes after it.
- `go test ./pkg/pod-utils/decorate/ ./pkg/config/` is clean.

Scope guard:
- Do not change how `scheduling_options.tolerations` is applied.
- Do not change the preset merge semantics in `pkg/config/jobs.go` beyond what a merge strategy needs.
```

### [validation] Validate preset affinity at config load
- where: `pkg/config/config.go:3124`
- necessity: could
- mode: deliver

```text
In `kubernetes-sigs/prow`, following PR #1058 "Add affinity rules to presets and allow presets to be overriden in jobs" (merged as 9f4d1868e46d0e0d4588bab8192813699544d859), add config-load validation for the new `presets[].affinity` surface.

What and where:
- `pkg/config/config.go`, `validatePodSpec` (~line 3124), which currently validates containers, env, volumes, and mounts but not affinity.
- Affinity is merged into `PodSpec.Affinity` in `pkg/config/jobs.go` (`mergeAffinity`).
- Existing `validatePodSpec` table tests are in `pkg/config/config_test.go`, `TestValidatePodSpec` (~line 1442).

Task:
1. Validate affinity invariants that the Kubernetes API server rejects, at minimum:
   - preferred scheduling terms: `weight` in 1..100 and a non-empty preference (node selector term / pod affinity term);
   - pod affinity/anti-affinity terms: non-empty `topologyKey`; required selector fields present;
   - node selector requirements: valid operators and non-empty values where required.
2. Return aggregated errors in the same style as the existing `validatePodSpec` errors.
3. Add table-driven tests for valid and invalid affinity, including a preset applied to a job.

Acceptance criteria:
- Invalid affinity in a job or preset fails config validation (`checkconfig`) instead of reaching the API server.
- Existing valid configs still load unchanged.
- `go test ./pkg/config/... ./cmd/checkconfig/...` is clean.

Scope guard:
- Do not change affinity merge semantics or add new config fields.
- Do not validate fields the API server accepts as optional beyond the listed invariants.
```

### [tests] Probe the affinity/override merge paths
- where: `pkg/config/jobs.go:77-200`, `pkg/config/config.go:3025-3067`
- necessity: could
- mode: probe

```text
In `kubernetes-sigs/prow`, following PR #1058 "Add affinity rules to presets and allow presets to be overriden in jobs" (merged as 9f4d1868e46d0e0d4588bab8192813699544d859), run a throwaway test probe of the preset affinity/override merge paths to hunt for real bugs. This is a probe: the tests are a means, not the deliverable.

What and where:
- `pkg/config/jobs.go`: `applyPreset`, `checkPresetOverride`, `mergeAffinity`.
- `pkg/config/config.go`: `resolvePresets`, `applyPresetWithIncludes`.
- Add tests in `pkg/config/jobs_test.go` (or a scratch `_test.go` file) using `resolvePresets` directly.

Behaviours to exercise:
1. Both job and preset set `PodAffinity` (required and preferred) — verify the exact resulting terms.
2. Job sets required node affinity and two matching presets also set it — verify job wins, no error, and the override is logged.
3. A diamond include graph (two matching presets include the same base preset carrying env, affinity, and a toleration) — verify no duplication or silent override.
4. Calling `resolvePresets` twice on the same `PodSpec` — verify whether preferred/pod affinity terms and tolerations duplicate or whether preset values are reclassified as job values.
5. A job and preset with the same volume mounted at different paths — verify which mount survives.
6. Job + preset env/volume/volume-mount collisions plus a second preset collision — verify the error path and message.

Triage every failure: is it a real implementation bug, or a wrong expectation/test mistake? If real bugs are found, apply the minimal fix (and report each failing case). If none, change no non-test files.

Acceptance criteria:
- Every behaviour above is exercised, and each failure is classified with evidence.
- If real bugs were found: the minimal fix is applied, with the failing case reported so it can be kept as a regression test.
- If none: no files changed.

Scope guard:
- Delete the probe tests before finishing; the deliverable is bug findings (and minimal fixes), not tests.
- Do not fix test-only problems, style, or unrelated code.
- Do not publish anything (no push, no GitHub comments).
```

### [performance] Take the pod-spec snapshot lazily
- where: `pkg/config/config.go:3029`
- necessity: could
- mode: deliver

```text
In `kubernetes-sigs/prow`, following PR #1058 "Add affinity rules to presets and allow presets to be overriden in jobs" (merged as 9f4d1868e46d0e0d4588bab8192813699544d859), avoid the unconditional pod-spec deep copy in `resolvePresets` when no preset matches.

What and where:
- `pkg/config/config.go`, `resolvePresets` (~line 3025): `job := spec.DeepCopy()` runs for every job before matching presets.
- The snapshot is threaded through `applyPresetWithIncludes`/`applyPreset` in `pkg/config/jobs.go`.
- Tests: `pkg/config/jobs_test.go` (`TestMergePreset`, `TestMergePresetAffinity`).

Task:
1. Take the pre-preset snapshot lazily, only after the first preset matches (and before applying it), so jobs matching no preset pay no deep copy.
2. Thread the snapshot through the existing helpers without changing their behavior; keep the "job/pre-preset values win" semantics identical.
3. Keep or add a test that proves matching presets still see the original (pre-preset) values.

Acceptance criteria:
- No deep copy occurs when `presets` is empty or when no preset matches.
- All existing merge/override tests pass unchanged.
- `go test ./pkg/config/` is clean.

Scope guard:
- Do not change merge semantics, error messages, or logging.
- Do not add new config fields or touch unrelated defaulting paths.
```
