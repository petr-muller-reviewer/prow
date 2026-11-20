---
pr: kubernetes-sigs/prow#1080
title: "WIP: Refresh queued PR presubmits before they start"
head_sha: e205f0d5065fe10535d236ba6d3f71048c23001c
base: main
reviewed_at: 2026-11-20T00:47:57Z
verdict: request-changes
---

# Review

## Verdict

Request changes: two confirmed, production-impacting issues were flagged independently by all three reviewers (code quality, maintainability, deployment risk) — the refresh is applied to non-GitHub refs, so every queued Gerrit presubmit fails, and any transient GitHub error terminally fails a required presubmit. The core design also needs an explicit decision on Tide serial jobs, pinned-base reruns, and job-definition skew.

The mechanics themselves are sound and well tested: the refresh sits in the no-pod branch of `syncTriggeredJob`, runs after `canExecuteConcurrently`, persists the refreshed refs before pod creation, and the pod's `JOB_SPEC` is verified by both a unit test and an end-to-end test. The problems are the eligibility predicate (too broad), the failure policy (terminal on every error), and the semantics of silently rewriting `Spec.Refs.BaseSHA` for jobs that were deliberately pinned or are owned by another component.

## What this PR does

- Gives Plank a GitHub client (`AddWithGitHubClient`) and constructs it in `prow-controller-manager`.
- When a `Triggered` presubmit is eligible to start (concurrency allows, no pod yet), looks up the current `heads/<BaseRef>` SHA and compares it with `Spec.Refs.BaseSHA`.
- On a difference, patches `Spec.Refs.BaseSHA`/`BaseLink` and requeues, so the pod is created in a later reconcile from the persisted refs.
- On a lookup error, marks the ProwJob `ErrorState` and completes it.
- Adds unit tests, an end-to-end integration test, and a `git/refs` handler to the integration fake GitHub server.

## Findings

### [blocking] Gerrit (non-GitHub) presubmits are failed by the GitHub lookup
- where: `pkg/plank/reconciler.go:819-825`
- concern: The predicate only checks `Type == PresubmitJob` and non-empty `Pulls`/`BaseRef`. Gerrit presubmits satisfy all three: `Refs.Org` is the Gerrit host (`pkg/gerrit/adapter/adapter.go:421`), they default to the Kubernetes agent (`pkg/config/config.go:3335`), and Plank consumes them. `GetRef` then builds `/repos/<gerrit-host>/...`, gets a 404, and the error path marks every queued Gerrit presubmit `ErrorState`. Flagged as critical by all three reviewers. Note that gating on `Refs.CloneURI != ""` is not sufficient — GitHub jobs can set `clone_uri` via job config — so use a provider-aware signal or move the refresh to a provider-aware component.
- excerpt: |
    if r.githubClient == nil || pj.Spec.Type != prowv1.PresubmitJob || pj.Spec.Refs == nil || len(pj.Spec.Refs.Pulls) == 0 || pj.Spec.Refs.BaseRef == "" {
        return false, nil
    }

    refs := pj.Spec.Refs
    baseSHA, err := r.githubClient.GetRef(refs.Org, refs.Repo, "heads/"+refs.BaseRef)

### [blocking] Any GitHub error terminally fails a required presubmit
- where: `pkg/plank/reconciler.go:742-753`
- concern: A rate limit, 5xx, auth problem, or network blip sets `ErrorState` + `SetComplete()`, and crier reports that as a red required check. A GitHub incident can therefore block Tide merges cluster-wide, and authors must repush. Prow normally retries non-terminal reconcile errors; use `github.IsNotFound` to distinguish a genuinely missing branch and requeue/back off for the rest. Flagged by all three reviewers.
- excerpt: |
    updated, err := r.refreshQueuedPresubmitBase(pj)
    if err != nil {
        pj.Status.State = prowv1.ErrorState
        pj.SetComplete()
        pj.Status.Description = fmt.Sprintf("Failed to refresh presubmit base SHA: %v", err)

### [should-fix] Tide serial presubmits are refreshed; Tide can accept the result for the new base
- where: `pkg/plank/reconciler.go:819`
- concern: The PR body says Tide behavior is unchanged, but only Tide batches are unaffected (they are `BatchJob` and skipped by the type check). Tide serial runs are `PresubmitJob` with `created-by-tide=true` and are refreshed. Crier stamps the refreshed `Refs.BaseSHA` into the status description (`pkg/github/report/report.go:90`), which is what Tide reads to decide whether a context passed for the current base — so a job generated from the old base's config can be accepted for the new base. Either skip Tide-labelled jobs or state that this change intentionally covers them.
- excerpt: |
    // pkg/tide/tide.go
    pj.Labels[kube.CreatedByTideLabel] = "true"

    // pkg/github/report/report.go
    Description: config.ContextDescriptionWithBaseSha(pj.Status.Description, refs.BaseSHA),

### [should-fix] Only the base SHA is refreshed; the job definition is not
- where: `pkg/plank/reconciler.go:834-837`
- concern: The ProwJob spec (commands, podspec, in-repo `.prow.yaml`-derived fields) is captured at trigger time. After the refresh the pod runs the new base's code with the old job definition, and the result is reported for the new base. If the new base renamed a make target or changed the command, the run can fail for a bogus reason or pass without exercising what the new commit intends. This is the concern raised in issue #1078 and needs a decision (opt-in per repo/job, compare the job definition, or document the limitation).
- excerpt: |
    refs.BaseSHA = baseSHA
    refs.BaseLink = fmt.Sprintf("%s/commit/%s", refs.RepoLink, baseSHA)

### [should-fix] Deliberately pinned bases are silently rebased, with no record of the original
- where: `pkg/plank/reconciler.go:832-836`
- concern: `Deck` reruns reuse the original `pj.Spec.Refs` and require `BaseRef` (`cmd/deck/rerun.go:265`), so rerunning an old job now runs its old head on today's base instead of reproducing it; `mkpj --base-ref X --base-sha Y` is affected too. `Spec.Refs.BaseSHA` becomes mutable after creation with no annotation preserving the trigger-time value and no opt-out. Decide the contract explicitly and add a pin/annotation, or call out the semantic change.
- excerpt: |
    prowJobSpec, labels, annotations, err := getProwJobSpec(pj.Spec.Type, cfg, pj.Spec.Job, pj.Spec.Refs, pj.Labels)

### [should-fix] PCM now implicitly requires GitHub credentials and an endpoint
- where: `cmd/prow-controller-manager/main.go:218`
- concern: `prow-controller-manager` has never needed GitHub flags (`test/integration/config/prow/cluster/prow_controller_manager_deployment.yaml` passes none) and `Validate()` sets `AllowAnonymous = true` (`cmd/prow-controller-manager/main.go:88`). Without `--github-endpoint`/`--github-token-path`, GHE refs and private repos 404, and public repos exhaust the 60 req/h anonymous budget almost immediately at scale. There is also no kill switch short of a rollback, and setting `--github-token-path` to an unmounted file now makes PCM fail to start. Operators need this called out; consider startup validation and a hot-reloadable disable.
- excerpt: |
    githubClient, err := o.github.GitHubClient(o.dryRun)
    if err != nil {
        logrus.WithError(err).Fatal("Failed to construct GitHub client for Plank")
    }

### [should-fix] GitHub call in the reconcile hot path without context propagation
- where: `pkg/plank/reconciler.go:818-823`
- concern: `refreshQueuedPresubmitBase` takes no context and `github.Client.GetRef` uses `context.Background()` (`pkg/github/client.go:3703`) with a 5-minute request timeout and 8 retries (`pkg/github/client.go:423-431`). With 10 reconcile workers (`pkg/plank/reconciler.go:110`) a degraded GitHub can pin workers and stall all ProwJob reconciliation, not just pod creation. Thread `ctx` through and use `GetRefWithContext`.
- excerpt: |
    func (r *reconciler) refreshQueuedPresubmitBase(pj *prowv1.ProwJob) (bool, error) {
        ...
        baseSHA, err := r.githubClient.GetRef(refs.Org, refs.Repo, "heads/"+refs.BaseRef)

### [should-fix] Integration fake server and test setup introduce hidden coupling
- where: `test/integration/cmd/fakeghserver/main.go:79,98-104`, `test/integration/test/plank_base_refresh_test.go:174-233`
- concern: `refHandler` returns `fakegithub.TestRef` for every ref of every repo, and the new test rewrites the shared `prow-controller-manager` deployment and waits for two full rollouts. Every other component already points at `fakeghserver`; if PCM's static integration config ever does too, the parallel `pod-utils` presubmit tests would have `fooHEADsha` rewritten to `abcde` and clonerefs would fail. The negative assertion also relies on a fixed `time.Sleep(2s)`.
- excerpt: |
    return fmt.Sprintf(`{"ref":"refs/%s","object":{"sha":"%s"}}`, vars["ref"], fakegithub.TestRef), http.StatusOK, nil

### [nit] `BaseLink` format is hardcoded and assumes a non-empty `RepoLink`
- where: `pkg/plank/reconciler.go:836`
- concern: GitHub trigger jobs use `/commit/`, Tide uses `/tree/` (`pkg/tide/github.go:560`), and `mkpj` leaves `RepoLink` empty (`cmd/mkpj/main.go:65-73`), producing `/commit/<sha>`. Guard on `RepoLink != ""` or reuse the existing ref formatting.
- excerpt: |
    refs.BaseLink = fmt.Sprintf("%s/commit/%s", refs.RepoLink, baseSHA)

### [nit] Duplicate entry points and post-construction dependency injection
- where: `pkg/plank/reconciler.go:107-116,155-157`
- concern: `Add` silently disables the feature via a nil client while `AddWithGitHubClient` enables it, and `add()` is now a 12-parameter function. `r.githubClient` is assigned after `newReconciler` rather than passed in. A single entry point with an options struct (and a one-method `refGetter` interface instead of the full `github.Client`) would age better.
- excerpt: |
    func AddWithGitHubClient(...) error {
        return add(..., "", ghc)
    }

### [nit] Tests encode implementation details and miss the risky paths
- where: `pkg/plank/reconciler_base_test.go:253-255`
- concern: Asserting `GetRef` was called exactly twice fails on an obvious improvement (one call, start in the same pass). There is no case for Gerrit-shaped refs, Tide-labelled jobs, or `podExists` (where the refresh must not run) — the non-GitHub omission is why the main regression is invisible to the suite.
- excerpt: |
    if baseClient.calls != 2 {
        t.Errorf("GetRef called %d times, want once for each pod-free reconciliation", baseClient.calls)
    }

## Checked

- The refresh runs only in the `podExists == false` branch of `syncTriggeredJob`, after `canExecuteConcurrently`; concurrency is re-evaluated on the requeued reconcile.
- The refreshed `Refs.BaseSHA` is persisted before the pod is built, and the pod's `JOB_SPEC` carries it; verified by both new tests.
- `Patch(..., MergeFrom(prevPJ))` is a narrow spec patch; the ProwJob CRD has no spec-immutability validation.
- `pjutil.TerminateOlderJobs` digests refs with SHAs excluded, so rewriting `BaseSHA` does not affect duplicate-job detection.
- `GetRef("heads/<branch>")` matches existing callers (tide, statusreconciler, mkpj); GETs are permitted in the GitHub client's dry-run mode.
- Tide `BatchJob`s are genuinely unaffected (type check); running/pending pods, periodics, and postsubmits are untouched.
- Built and tested locally: `gofmt`, `go vet`, `go build ./cmd/prow-controller-manager ./pkg/plank`, `go test ./pkg/plank/...`.

## Open questions

- Should this be opt-in (per repo/job annotation or a `Plank` config knob) rather than on by default for every presubmit?
- Should the refresh live in the trigger path (which knows it is GitHub and can also refresh the job definition) instead of Plank?
- Is a bounded number of refresh attempts worth adding, so a branch that moves during every ~1s requeue cannot keep resetting a job indefinitely?
- For Deck reruns and `mkpj` pinned bases, is the intent to reproduce the original run or retest against the current base?
