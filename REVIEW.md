---
pr: kubernetes-sigs/prow#804
title: "plank: revive pods terminated by kubelet"
head_sha: e039405950fc1f0303e1388925b839aab7016f60
base: main
reviewed_at: 2026-08-11T11:28:56Z
verdict: needs-discussion
refresh_log:
  - from_sha: e039405950fc1f0303e1388925b839aab7016f60
    to_sha: e039405950fc1f0303e1388925b839aab7016f60
    at: 2026-08-11T11:28:56Z
    summary: No code changes. Prucek softened his objection, saying he'd be ok approving if the feature were opt-in (per dobesv's earlier suggestion) and pulled in maintainers petr-muller and smg247 for input. petr-muller replied he likes the feature at a high level and wouldn't mind merging it, but hadn't been closely engaged since others seemed to have it covered. First sign of convergence toward a path forward (opt-in) since the correctness/scope debate stalled; lgtm still absent, smg247 has not yet responded.
  - from_sha: 937cca0dd5d26c70a8601af8b1748c4b8c4ddf9f
    to_sha: e039405950fc1f0303e1388925b839aab7016f60
    at: 2026-08-09T14:05:46Z
    summary: Head SHA changed but only via a merge from main picking up unrelated commits (dependabot bumps, testfreeze plugin change) — no changes under pkg/plank or pkg/pjutil, the PR's actual feature is untouched. Comment activity - yuluo-yx asked mimowo/Prucek for next steps since consensus hasn't been reached; mimowo replied that the original flakiness (spot VM CI jobs) is no longer occurring since they stopped using spot VMs, and proposed keeping the PR open one more week before closing as lazy consensus if no maintainer moves it forward. Design/correctness debate remains unresolved; lgtm still absent.
  - from_sha: 937cca0dd5d26c70a8601af8b1748c4b8c4ddf9f
    to_sha: 937cca0dd5d26c70a8601af8b1748c4b8c4ddf9f
    at: 2026-07-25T12:51:35Z
    summary: No code changes. Maintainer review-comment thread on reconciler.go:724 shows mimowo proposing to generalize the fix from TerminationByKubelet-only to any DisruptionTarget reason (rename to podWasDisrupted/PodUnexpectedStopCauseDisruption) and asking the author to broaden scope per a testing-leads Slack discussion; droslean pushed back, noting Prow isn't Kubernetes-only and wants to check OpenShift's node-status handling before proceeding. Author has not yet responded substantively (still needs to join the Slack channel). Verdict changed from approve to needs-discussion pending resolution of this design disagreement.
  - from_sha: 937cca0dd5d26c70a8601af8b1748c4b8c4ddf9f
    to_sha: 937cca0dd5d26c70a8601af8b1748c4b8c4ddf9f
    at: 2026-07-27T10:17:44Z
    summary: No code changes. Maintainer Prucek raised a correctness objection and canceled lgtm ("/lgtm cancel"), arguing kubelet terminates pods for many legitimate non-retriable reasons (resource pressure, graceful shutdown, node-level preemption/OOM) and that treating all kubelet-initiated terminations as retriable is the wrong direction. mimowo rebutted with upstream k8s docs stating the DisruptionTarget condition is specifically only attached for retriable disruption scenarios, and is deliberately omitted for pod-caused disruptions that would recur on retry. lgtm label is now removed. Thread unresolved as of this refresh.
  - from_sha: 937cca0dd5d26c70a8601af8b1748c4b8c4ddf9f
    to_sha: 937cca0dd5d26c70a8601af8b1748c4b8c4ddf9f
    at: 2026-07-29T00:44:09Z
    summary: No code changes. Prucek narrowed the objection - concedes the DisruptionTarget design intent (retry makes sense for spot reclamation/graceful shutdown) but argues TerminationByKubelet also covers node-pressure eviction (memory/disk/PID pressure), where retrying could worsen an already-pressured cluster. mimowo countered that MaxRevivals=3 already bounds this, and that relying on humans to manually diagnose pressure-vs-infra causes before retrying is unrealistic (contributors just click retry regardless). Author yuluo-yx posted a 3-point summary of the debate and reframed the open question as whether Prow should auto-recover from infra disruption by default, and how to add granular per-cause/per-cluster control. No resolution yet; lgtm still absent.
---

## Summary

Adds `PodUnexpectedStopCauseTerminationByKubelet`: pods with `status.reason == "Terminated"` and a `DisruptionTarget` condition (`status: True`, `reason: TerminationByKubelet`) are now routed into the existing unexpected-stop/revival path in `syncPendingJob` instead of being treated as a genuine test failure. Bounded by the existing `MaxRevivals` limit. Fixes kubernetes-sigs/kueue#12856.

Reviewed from three additional maintainer perspectives (code quality, maintainability, deployment risk) via parallel sub-agents; all three independently returned APPROVE. Findings below merge the direct review with that panel's output.

**Since previous review** (no code changes; `head_sha` unchanged):
- Maintainer `mimowo` opened a design discussion on `reconciler.go:724` proposing to generalize the check to any `DisruptionTarget` condition (not just `TerminationByKubelet`), renaming to `podWasDisrupted`/`PodUnexpectedStopCauseDisruption`, and asked the author to broaden the PR's scope per an off-thread Slack conversation with testing leads.
- Maintainer `droslean` objected: Prow isn't Kubernetes-only, and generalizing needs investigation of how OpenShift represents node/pod disruption before it's safe to default this behavior.
- Author `yuluo-yx` has not yet substantively responded (hasn't joined the referenced Slack channel yet).
- Unrelated informational comment from `dobesv` describing an alternative approach (a separate controller instead of a core Prow change) for the same underlying spot-instance problem.
- Maintainer `Prucek` raised a correctness objection and ran `/lgtm cancel`, disputing that kubelet-initiated terminations are generally retriable (cites resource pressure, graceful node shutdown, node-level preemption, node-level OOM as legitimate non-retriable kubelet termination reasons). `mimowo` rebutted by quoting upstream k8s docs: the `DisruptionTarget` condition is documented as being attached specifically and only for retriable disruption scenarios, and is deliberately withheld for pod-caused disruptions that would reoccur on retry — i.e., the PR's premise (a pod with this specific condition+reason is safe to retry) appears to match documented k8s semantics. `lgtm` label has been removed; thread is unresolved.
- `Prucek` narrowed the objection: concedes the retriable-by-design intent for spot reclamation/graceful shutdown, but points out `TerminationByKubelet` also covers node-pressure eviction (memory/disk/PID pressure), where retrying could add load to an already-pressured cluster. `mimowo` countered that `MaxRevivals=3` already bounds the blast radius, and that relying on contributors to manually diagnose pressure-vs-infra causes before deciding to retry is unrealistic. Author `yuluo-yx` posted a 3-point summary reframing the open question as: should Prow auto-recover from infra disruption by default, and how to add granular per-cause/per-cluster control. Still unresolved; `lgtm` remains absent.
- (2026-08-09) `head_sha` changed to `e039405` via a merge from `main`, but the merge only pulls in unrelated commits (dependabot dependency bumps, a `testfreeze` plugin change) — no changes under `pkg/plank` or `pkg/pjutil`, so the feature diff is unchanged and none of the prior code findings are affected.
- `yuluo-yx` asked `mimowo`/`Prucek` what the next steps are, noting consensus hasn't been reached. `mimowo` replied that the original motivating flakiness (spot-VM CI jobs) no longer occurs since they moved off spot VMs for CI, and proposed leaving the PR open one more week, closing it as lazy consensus if no maintainer moves it forward. Design/correctness debate (retriable-by-design vs. node-pressure over-inclusion) remains unresolved; `lgtm` still absent.
- (2026-08-10) `Prucek` signaled he'd be willing to approve if the feature were made opt-in (echoing `dobesv`'s earlier separate-controller-flavored suggestion), and asked maintainers `petr-muller` and `smg247` to weigh in. `petr-muller` responded favorably at a high level ("would not mind merging it") but noted he hadn't been closely tracking the thread. This is the first concrete move toward a compromise (gate the behavior behind an opt-in, addressing the earlier "no opt-out for this revival cause" finding below) since the correctness/scope debate stalled; `smg247` has not yet responded and `lgtm` remains absent.

## Findings

### [question] lgtm canceled: dispute over whether kubelet terminations are generally retriable
- where: PR-level discussion (issue comments, 2026-07-27)
- concern: `Prucek` canceled `lgtm`, arguing the kubelet terminates pods for multiple legitimate, non-retriable reasons (resource-pressure eviction, graceful node shutdown, node-level preemption, node-level OOM), and objects to treating all kubelet-initiated terminations as retriable infrastructure glitches. `mimowo` responded citing upstream Kubernetes documentation: the `DisruptionTarget` pod condition is specifically and only attached for retriable disruption scenarios; pod-caused disruptions that would recur on retry (e.g. exceeding container resource limits) deliberately do NOT receive this condition. This directly addresses Prucek's concern if accurate — the PR doesn't match on "any kubelet termination," only ones additionally flagged via `DisruptionTarget`/`TerminationByKubelet`, which per docs excludes the non-retriable cases Prucek lists. As of this refresh, `Prucek` has not replied to the rebuttal and `lgtm` remains removed.
- excerpt: |
    Prucek: "Kubelet terminates pods for many legitimate reasons: Resource
    pressure ... Graceful node shutdown ... Priority-based preemption at the
    node level ... Node-level OOM ... I don't think we should be going in this
    direction. /lgtm cancel"

    mimowo: "...DisruptionTarget condition ... was designed specifically to
    mark retriable terminations ... In all other disruption scenarios, ...
    Pods don't receive the DisruptionTarget condition because the disruptions
    were probably caused by the Pod and would reoccur on retry."

### [question] narrowed objection: does TerminationByKubelet over-include node-pressure eviction?
- where: PR-level discussion (issue comments, 2026-07-27)
- concern: `Prucek` conceded the `DisruptionTarget` design intent (retriable-by-design for spot reclamation/graceful shutdown) but points out `TerminationByKubelet` specifically also fires for node-pressure eviction (memory/disk/PID pressure) — a case where retrying reproduces the same resource contention rather than escaping a transient infra event, and could compound cluster pressure. `mimowo` counters that `MaxRevivals=3` bounds the downside, and that expecting contributors to manually distinguish "safe to retry now" from "will hit the same pressure again" before clicking retry is unrealistic (most just retry blindly regardless of cause). Author `yuluo-yx` reframed this as a broader open design question: should Prow auto-recover from infra disruption by default, and is there a need for granular per-cause/per-cluster opt-in/opt-out (which would also address the earlier `[question] no opt-out for this revival cause, unlike eviction` finding below). Unresolved as of this refresh.
- excerpt: |
    Prucek: "...TerminationByKubelet covers both spot/graceful-node-shutdown
    and node-pressure eviction (memory, disk, PID pressure). For node-pressure
    eviction, the pod was terminated because the node ran out of resources —
    retrying is likely to hit the same problem again, and auto-reviving could
    make things worse by adding load to an already-pressured cluster."

    mimowo: "...the mechanism is anyway capped by max_revivals=3 ...
    contributors don't check if a PR failed maybe due to PID pressure on node
    ... most of them just randomly do "retry". Relying on humans being slow is
    not a good strategy."

    yuluo-yx: "...key questions are: Should Prow automatically recover from
    such infrastructure disruptions by default? And how can we provide
    sufficiently granular control for expensive tasks, different clusters, and
    various causes of disruption?"

### [question] scope disagreement: TerminationByKubelet-only vs. any DisruptionTarget
- where: `pkg/plank/reconciler.go:714-728` (`podWasTerminatedByKubelet`)
- concern: Maintainer `mimowo` proposed generalizing this to match any `DisruptionTarget` condition reason (covering preemption/eviction/kubelet-termination uniformly), citing upstream k8s docs that `DisruptionTarget` broadly signals k8s-initiated disruption, and asked the author to broaden the PR per a Slack discussion with testing leads (PR comment id 3641086578). Maintainer `droslean` objects (PR comment id 3644098145): Prow is not Kubernetes-only, and defaulting this behavior needs prior investigation of how OpenShift represents equivalent node/pod disruption. This is an open, unresolved design question that could change the shape of the fix (new type name `PodUnexpectedStopCauseDisruption`, broader condition match) before merge. The author has not yet responded substantively.
- excerpt: |
    // mimowo's proposed generalization (PR comment id 3638465860):
    if podWasDisrupted(pod) {
        // DisruptionTarget identifies a retriable disruption affecting the Pod,
        // such as preemption, eviction, or termination by the kubelet. Treat it as
        // unexpected so Plank can revive the Pod. This also covers spot node
        // reclamation observed in Kueue CI.
        return PodUnexpectedStopCauseDisruption
    }

### [should-fix] no test for kubelet-termination cause at MaxRevivals boundary
- where: `pkg/plank/controller_test.go` (TestSyncPendingJob table)
- concern: The table has a case for eviction hitting `PodRevivalCount >= MaxRevivals` ("don't delete evicted pod w/ revivalCount == maxRevivals, complete PJ instead") but no analogous case for `PodUnexpectedStopCauseTerminationByKubelet`. Flagged independently by the code-quality reviewer and corroborated by the deployment-risk reviewer's emphasis on `MaxRevivals` as the key safety bound — worth confirming the new cause participates correctly in the existing revival-limit path, not just cause detection.
- excerpt: |
    case pj.Status.PodRevivalCount >= *r.config().Plank.MaxRevivals:
        // MaxRevivals is reached, complete the PJ and mark it as errored.
        ...
        pj.Status.Description = fmt.Sprintf("Job pod reached max revivals (%d) after being stopped unexpectedly (%s)", pj.Status.PodRevivalCount, podUnexpectedStopCause)

### [nit] undocumented branch ordering in getPodUnexpectedStopCause
- where: `pkg/plank/reconciler.go:682-711`
- concern: The new kubelet-termination check is inserted between the `Evicted` and `Unreachable` checks with no comment on whether order is significant. Flagged independently by both the code-quality and maintainability reviewers — a future contributor adding another cause has to re-derive that the branches are mutually exclusive on `pod.Status.Reason` rather than order-dependent. A one-line comment would save that work.

### [nit] rationale comment lives on the caller, not the gating function
- where: `pkg/plank/reconciler.go:687-693` vs `pkg/plank/reconciler.go:714-728` (`podWasTerminatedByKubelet`)
- concern: The explanatory comment (spot-node example, kueue issue link) is attached at the `getPodUnexpectedStopCause` call site, but the actual gating logic — requiring both `Status.Reason == Terminated` and the specific condition/reason to agree — lives in `podWasTerminatedByKubelet` itself, where a reader jumping straight to the helper won't see it.

### [question] trailing "feat: add comment" commit
- where: git log (`937cca0dd`)
- concern: Second commit on the branch has an uninformative message but only adds the explanatory comment above `podWasTerminatedByKubelet`'s call site (verified via `git show`) — no functional change. Not blocking, but worth a squash/reword before merge for history clarity.

### [question] no opt-out for this revival cause, unlike eviction
- where: `pkg/plank/reconciler.go:487` (`PodUnexpectedStopCauseEvicted && pj.Spec.ErrorOnEviction`) vs the new `PodUnexpectedStopCauseTerminationByKubelet` branch
- concern: Eviction has a per-job opt-out (`ErrorOnEviction`) for teams that want fail-fast behavior instead of retry. The new kubelet-termination cause has no equivalent — any matching pod is always revived (subject to `MaxRevivals`). Flagged by the deployment-risk reviewer as worth raising with the author, not a merge blocker.

### [question] consider a Prometheus label for unexpected-stop-cause
- where: `pkg/plank/reconciler.go` (revival logging, e.g. line ~502)
- concern: `unexpected-stop-cause` is only a structured log field today; a metric/label would make revival-cause trends (including this new one) dashboard-visible without log-scraping. Applies to the whole enum, not unique to this PR — flagged as optional deferred cleanup by the maintainability reviewer.

## Checked
- `podWasTerminatedByKubelet` uses upstream `corev1.DisruptionTarget` / `corev1.PodReasonTerminationByKubelet` constants, not ad-hoc strings; confirmed present in vendored `k8s.io/api` (v0.29.6+).
- Check ordering in `getPodUnexpectedStopCause` (evicted → kubelet-terminated → node-unreachable) has no overlap risk; `pod.Status.Reason` values are mutually exclusive across these paths.
- Loop over `pod.Status.Conditions` correctly returns on first `DisruptionTarget` match (k8s only sets one condition per type).
- `go build ./pkg/plank/...`, `go vet`, and `go test ./pkg/plank/... -run 'TestGetPodUnexpectedStopCause|TestSyncPendingJob'` pass locally at head SHA.
- New `TestGetPodUnexpectedStopCause` table covers happy path + 4 negative cases (inactive condition, wrong reason, no condition, condition present but pod not Terminated).
- `TestSyncPendingJob` renamed existing case for clarity and added a new case confirming revival (pod deleted, job stays pending) rather than failure.
- No config/CRD/struct-tag changes; behavior is unconditional post-upgrade, inert on pre-1.26 clusters lacking `PodDisruptionConditions`; rollback is a plain binary revert.
- No RBAC, dependency, or resource-consumption changes; reuses the existing delete/recreate pod flow.

## Open questions
- Was a max-revivals test case for this new cause intentionally omitted, or just missed alongside the eviction one?
- Any interest in squashing the two commits before merge, given the second only adds a comment?
- Is an `ErrorOnEviction`-style opt-out warranted for teams with strict-SLA jobs that prefer fail-fast over retry on kubelet-initiated termination?
- Should this behavior change (retry instead of immediate failure for kubelet-terminated pods) be called out explicitly in release notes, given it silently affects fail-fast signals for operators on spot/preemptible infrastructure?
