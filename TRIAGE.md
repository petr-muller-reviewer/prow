---
issue: kubernetes-sigs/prow#824
title: "Deck PR status page shows stale contexts for renamed or removed presubmit jobs."
state: open
labels: area/status-reconciler, area/deck
main_sha: c8b6c2ce8b130126a73672324a6ced4bd83df8a4
triaged_at: 2026-08-17T22:40:36Z
verdict: accepted
---

## Findings

### [cause] status-reconciler's retirement pass is one-shot with no re-visit mechanism
- detail: `pkg/statusreconciler` retires a removed/renamed presubmit's GitHub context once, synchronously, when the config delta that removed it is processed. There is no persisted record of which contexts/PRs were actually retired — only the whole config snapshot persists — so nothing can detect or catch up on missed work later. This produces two independent, confirmed blind spots below.
- evidence: `pkg/statusreconciler/controller.go:161-322`, `pkg/statusreconciler/status.go:73-95`

### [cause] In-flight job at retirement time is not aborted or rechecked
- detail: `retireRemovedContexts` retires a context immediately with no check against ProwJob state. `processPR` has no ProwJob lookup, and there is no abort/cancellation logic anywhere in `pkg/statusreconciler` (confirmed via grep, zero hits). A ProwJob still running when its job is removed from config can finish afterward and re-report its own (now stale) status via crier, silently overwriting the retirement.
- evidence: `pkg/statusreconciler/controller.go:294-322`, `pkg/statusreconciler/migrator/migrator.go:232-249`

### [cause] Closed-then-reopened PR is permanently missed
- detail: `GetPullRequests` hits `/repos/%s/%s/pulls` with no `state` parameter, defaulting to GitHub's `state=open`. `migrator.Migrate()` and `controller.go`'s `triggerNewPresubmits` both consume it this way. A PR closed at the moment a config delta is processed is invisible to that retirement pass; since each delta's diff only reports jobs that transitioned out of config in that specific delta, a later reopen of the PR is never revisited and its stale context is permanently missed.
- evidence: `pkg/github/client.go:2144-2169`, `pkg/statusreconciler/migrator/migrator.go:252-268`, `pkg/statusreconciler/controller.go:228`, `pkg/statusreconciler/controller.go:409-437` (`removedPresubmits`)

### [reproducibility] Third hypothesis investigated and retracted
- detail: An initially suspected gap — a stale context "spreading" to a brand-new PR/commit after retirement — was checked and ruled out. `getHeadContexts` only ever queries the current HEAD SHA via `GetCombinedStatus`/`ListCheckRuns`; there is no mechanism for an old commit's status to appear on an unrelated new commit.
- evidence: `pkg/prstatus/prstatus.go`

### [related-code] Sequential controller loop constrains any in-flight-job fix
- where: `pkg/statusreconciler/controller.go:161-183`
- excerpt: |
    func (c *Controller) Run(ctx context.Context) {
      changes, err := c.statusClient.Load()
      if err != nil {
        logrus.WithError(err).Error("Error loading saved status.")
        return
      }

      for {
        select {
        case change := <-changes:
          start := time.Now()
          log := logrus.WithField(...)
          if err := c.reconcile(change, log); err != nil {
            log.WithError(err).Error("Error reconciling statuses.")
          }
          log.WithField("duration", ...).Info("Statuses reconciled")
          c.statusClient.Save()
        case <-ctx.Done():
          logrus.Info("status-reconciler is shutting down...")
          return
        }
      }
    }
- relevance: `Run()` is single-goroutine and fully sequential — one delta's `reconcile` (trigger/retire/migrate) completes, then `statusClient.Save()` is called unconditionally, even if `reconcile` returned an error, before the next delta is read. Any fix that blocks mid-retirement (e.g. waiting for a job to finish) would stall every subsequent delta across every repo, not just the affected one.

### [related-code] Existing abort + crier-skip pattern a fix could reuse
- where: `pkg/plugins/trigger/pull-request.go:170-199`, `pkg/crier/reporters/github/reporter.go:72-87`
- excerpt: |
    // abortAllJobs (pull-request.go): sets Status.State/Status.Description only
    job.Status.State = prowapi.AbortedState
    job.Status.Description = abortedDescription
    // ShouldReport (github reporter.go): checks Spec.Report first
    func (c *Client) ShouldReport(_ context.Context, _ *logrus.Entry, pj *v1.ProwJob) bool {
      if !pj.Spec.Report {
        return false
      }
      ...
    }
- relevance: `abortAllJobs` does not touch `Spec.Report`. Crier's github reporter `ShouldReport` checks `Spec.Report` first and skips entirely if false. A status-reconciler abort that additionally sets `Spec.Report = false` in the same update would prevent a report race with crier without needing direct coordination — verified both sides of this claim in code.

### [related-code] ProwJobClient reaches the controller but not the migrator
- where: `pkg/statusreconciler/controller.go:41-70`
- excerpt: |
    func NewController(..., prowJobClient prowv1.ProwJobInterface, githubClient github.Client, ...) *Controller {
      ...
      return &Controller{
        ...
        prowJobTriggerer: &kubeProwJobTriggerer{
          prowJobClient: prowJobClient,
          githubClient:  githubClient,
          ...
        },
        githubClient: githubClient,
        statusMigrator: &gitHubMigrator{
          githubClient:    githubClient,
          continueOnError: continueOnError,
        },
        ...
      }
    }
- relevance: `prowJobClient` is passed into `NewController` and used only to build `kubeProwJobTriggerer` (the presubmit-triggering path); `gitHubMigrator` (the retirement/migration path) is constructed with only `githubClient`, no ProwJob awareness at all. A fix needs to thread `prowJobClient` into `gitHubMigrator` as well. `ProwJobSpec.Context` is a plain string field (`pkg/apis/prowjobs/v1/types.go:171`), directly usable for matching without any annotation scheme.

### [related-pr] kubernetes-sigs/prow#615
- ref: kubernetes-sigs/prow#615
- relevance: Prior attempt at a Deck-side fix for the originally-reported symptom (drop GitHub-only contexts client-side). Rejected in review by ivankatliarchuk for silently hiding non-Prow required contexts; closed unmerged by the stale bot. Superseded once the issue was reframed around status-reconciler; retained as evidence against re-attempting a client-side hide.

### [related-issue] kubernetes/test-infra#36399
- ref: kubernetes/test-infra#36399
- relevance: Likely origin of the "please file in kubernetes-sigs/prow" redirect that led to #824. Describes a related-but-distinct symptom (stale results in the prow bot's PR comment, attributed to delta-vs-full-recompute state), not the status-reconciler retirement gaps identified here.

## Checked
- Independently re-verified every technical claim from the issue thread's 2026-08-14 comment against the actual code (not trusted as-is). All confirmed except one: the claim that a follow-up `GetPullRequest` call is needed per match is inaccurate — `Migrate()` already receives full `PullRequest` structs (head SHA, base ref) from `GetPullRequests` and uses them directly.
- Confirmed `Controller.Run()`'s sequential single-channel model (relevant to any blocking-vs-abort design for the in-flight-job fix).
- Checked `pkg/statusreconciler/controller_test.go` and `pkg/statusreconciler/migrator/migrator_test.go` — no existing coverage for ProwJob running-state, aborts, or closed/reopened PRs; both use a consistent table-driven + interface-fake pattern a fix would extend.
- Confirmed upstream `main` has not moved on `pkg/statusreconciler/*` or `cmd/deck/static/pr/pr.ts` since the prior (2026-08-13) triage's anchor SHA.
- Re-read the full issue thread (body + all comments through 2026-08-14) and PR #615 (body, comments, reviews) to confirm the scope shift from Deck to status-reconciler is accurate and not a misreading.

## Next steps
- Retitle the issue (or add a clarifying top comment) to reflect the actual scope: status-reconciler retirement gaps, not a Deck display bug.
- Consider removing the `area/deck` label — the Deck-side proposal was explicitly rejected on the issue, and research confirms the real gaps are entirely within status-reconciler.
- Split into two independently-shippable pieces given very different risk profiles: (1) closed-then-reopened-PR revisit cursor, (2) in-flight-job abort.
- Before any implementation on the in-flight-job piece, get explicit maintainer sign-off on the abort-vs-block trade-off (sunk CI compute vs. a residual report-race window) — this is a product decision, not an implementation detail.
- Once cursor-persistence semantics (per-repo scoping, only-advance-on-full-success) are agreed, the closed-PR piece could be scoped as a standalone, lower-effort follow-up issue.

## Open questions
- Should status-reconciler abort an in-flight job whose context is being retired (accepting sunk CI compute cost), or leave it running (accepting the residual race where its late report could overwrite the retirement)?
- Should the closed-PR-revisit cursor be scoped globally or per-repo, and should it only advance after a fully successful pass, to avoid silently reproducing the same "missed retirement" bug class it's meant to fix?
- Is there an existing Prow pattern for rate-limit-aware GitHub Search API batching across many repos in one config delta, or would this be the first such use?
- Should these two gaps be tracked and worked on as one issue, or split into separate issues given their very different effort levels?
