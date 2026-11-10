---
pr: kubernetes-sigs/prow#931
title: "chore(deps): bump sigs.k8s.io/controller-runtime from 0.24.1 to 0.25.0 in the kubernetes group across 1 directory"
head_sha: 450cf57da53d1d3412fcde76945f4f45cc63dea1
base: main
reviewed_at: 2026-11-10T01:14:34Z
verdict: approve
refresh_log:
  - at: 2026-11-10T01:14:34Z
    old_sha: 450cf57da53d1d3412fcde76945f4f45cc63dea1
    new_sha: 450cf57da53d1d3412fcde76945f4f45cc63dea1
    summary: "No code changes; approval and merge recorded, pending-CI condition satisfied."
---

## Verdict

Approve. This dependency-only update aligns controller-runtime with Prow's existing Kubernetes v0.37 direct dependencies. The release is 11 days old and has broad production use here, but its breaking and new behavior does not intersect with Prow's imported APIs. The review's merge condition was met: the PR was approved and merged on 2026-09-14.

## What this PR does

- Updates the direct controller-runtime dependency from v0.24.1 to v0.25.0.
- Updates its indirect k8s.io/apiextensions-apiserver dependency from v0.36.3 to v0.37.0.
- Refreshes the corresponding module checksums only.
- Does not modify Prow source, configuration, or tests.

Since previous review:

- No code changes: the PR head is still `450cf57da53d1d3412fcde76945f4f45cc63dea1`.
- `petr-muller` approved the PR at 2026-09-14T22:03:35Z; the approval-notifier bot recorded the approval at 2026-09-14T22:04:13Z.
- The PR was merged into `main` at 2026-09-14T22:23:20Z, satisfying the pending-CI condition.

## Findings

None.

## Resolved

- The merge condition in Open questions was satisfied: the PR merged at 2026-09-14T22:23:20Z.

## Checked

- `go.mod:83` declares controller-runtime directly; it is imported by 55 Go files (30 production and 25 test files), covering ProwJob reconcilers, managers, caches, clients, and fake-client tests.
- `go.mod:166` now aligns the transitive apiextensions-apiserver version with Prow's existing direct Kubernetes v0.37.0 dependencies.
- controller-runtime v0.25.0 is a tagged release published 2026-09-03 (11 days before review); the v0.37.0 Kubernetes release was published 2026-08-26.
- Upstream changes include opt-in read-your-writes cache consistency, fake-client enhancements and fixes, controller/cache shutdown fixes, and the Kubernetes v1.37 dependency update. Prow does not use the new read-your-writes, typed informer, EventRecorder, PriorityQueue, or affected fake-client APIs.
- GitHub's published security-advisory endpoint returned no controller-runtime advisory. The PR changes only `go.mod` and `go.sum`.

## Open questions

None.

## Dependency followups

Followups accepted from the `sigs.k8s.io/controller-runtime` v0.24.1→v0.25.0 changelog
(PR #931, merged as `d91e0f84e0f2050b41390b28b69a05f5284ba4a2`).

Examined with no action: `k8s.io/apiextensions-apiserver` v0.36.3→v0.37.0 (indirect, not
imported); `GetEventRecorderFor` deprecation (the only new deprecation in range, not called
by Prow); `TypedInformer` source (does not supersede `source.Kind` usage in
`pkg/plank/reconciler.go:155`); webhook server `Port: -1` (Prow never starts a manager
webhook server); fakeclient `WithGlobalResourceVersionCounter` (no Prow test depends on
cross-object resourceVersion ordering).

### [simplification] Enable read-your-writes consistency and drop cache-wait polls

- unlock: `sigs.k8s.io/controller-runtime` v0.24.1→v0.25.0
- where: `pkg/plank/reconciler.go:659`, `pkg/plank/reconciler.go:775`,
  `pkg/plank/reconciler.go:882`, `pkg/crier/reporters/criercommonlib/updatereportstatus.go:52`
- necessity: could — experimental upstream feature, but it removes four timeout-based polls
  that exist solely to wait for the cache to observe a write.

```
In kubernetes-sigs/prow, now that PR #931 ("chore(deps): bump sigs.k8s.io/controller-runtime
from 0.24.1 to 0.25.0 in the kubernetes group across 1 directory", merged as
d91e0f84e0f2050b41390b28b69a05f5284ba4a2) bumped sigs.k8s.io/controller-runtime to v0.25.0,
adopt the new experimental read-your-writes cache consistency and remove Prow's hand-rolled
cache-wait loops.

What the bump offers: v0.25.0 adds client.Options.Cache.EnableReadYourWritesConsistency
(*bool, experimental). From controller-runtime v0.25.0 pkg/client/client.go:
"EnableReadYourWritesConsistency controls if read requests against the cache will block until
the cache observed all write requests that started before the read request. Lists will wait
for all pending write requests to the gvk the List is for." Individual requests can opt out
with client.DisableReadYourWritesConsistency.

Task:
1. Set EnableReadYourWritesConsistency (ptr.To(true)) on the cache-backed clients that
   currently need the polls:
   - cmd/crier/main.go: add Client: ctrlruntimeclient.Options{Cache:
     &ctrlruntimeclient.CacheOptions{EnableReadYourWritesConsistency: ptr.To(true)}} to the
     manager.Options (controller-runtime fills Cache.Reader when nil; see v0.25.0
     pkg/cluster/cluster.go).
   - cmd/prow-controller-manager/main.go: same for the plank manager.
   - pkg/flagutil/kubernetes_cluster_clients.go:365 (BuildClusters overrideOptions): set
     newOptions.Client.Cache.EnableReadYourWritesConsistency so build-cluster clients get it
     too; this covers the pod-create visibility wait.
2. Delete these now-redundant poll loops and their surrounding comments, keeping the
   returned errors meaningful:
   - pkg/plank/reconciler.go:659-671 (state-transition wait)
   - pkg/plank/reconciler.go:775-786 (MaxConcurrency/JobQueueName wait)
   - pkg/plank/reconciler.go:882-894 (pod-create visibility wait)
   - pkg/crier/reporters/criercommonlib/updatereportstatus.go:52-65 (report-state wait)
3. Check whether other components (cmd/sinker, cmd/tide, cmd/deck) have cache-read-after-write
   assumptions worth opting in for; enable only where such an assumption exists.

Acceptance criteria:
- go build ./... and go test ./pkg/plank/... ./pkg/crier/... ./pkg/flagutil/... pass.
- No wait.PollUntilContextTimeout remains at the four call sites.
- Behaviour is unchanged for state transitions, MaxConcurrency gating, pod creation, and
  report-state de-duplication (run test/integration/test if the environment allows).
- The PR description notes that read-your-writes consistency is experimental in v0.25.0 and
  may change in a future controller-runtime release.

Scope guard:
- Do not change unrelated controllers, retry backoffs, or the PriorityQueue code path.
- Do not adopt the feature for read-only clients or deck's read path.
- Do not upgrade controller-runtime further; stay on v0.25.0.
```

### [new-feature] Opt into client-go REST client metrics

- unlock: `sigs.k8s.io/controller-runtime` v0.24.1→v0.25.0
- where: `pkg/metrics/metrics.go:64` (the scrape handler that already merges
  `ctrlruntimemetrics.Registry`)
- necessity: could — observability only; adds API-request latency/size/retry series to every
  component that uses `pkg/metrics`.

```
In kubernetes-sigs/prow, now that PR #931 ("chore(deps): bump sigs.k8s.io/controller-runtime
from 0.24.1 to 0.25.0 in the kubernetes group across 1 directory", merged as
d91e0f84e0f2050b41390b28b69a05f5284ba4a2) bumped controller-runtime to v0.25.0, expose the
newly opt-in client-go REST client metrics through Prow's metrics endpoint.

What the bump offers: controller-runtime v0.25.0 changed pkg/metrics so that only
rest_client_requests_total is registered by default; the others are opt-in via
metrics.RegisterRESTClientMetrics(metrics ...RESTClientMetric) or
metrics.RegisterRESTClientMetricsWithOptions(RESTClientMetricsOptions{DurationBuckets: ...},
...). Available: MetricRequestLatency, MetricDNSResolutionLatency, MetricRequestSize,
MetricResponseSize, MetricRateLimiterLatency, MetricRequestRetry.

Task:
1. In pkg/metrics/metrics.go, register the opt-in metrics exactly once per process and after
   controller-runtime's metrics package init (e.g. a sync.Once helper called from
   ExposeMetricsWithRegistry, or a package init in this file).
2. Use RegisterRESTClientMetricsWithOptions with the Kubernetes-default buckets unless there
   is a concrete reason to override them; enable all six metrics initially.
3. No server wiring should be needed: pkg/metrics/metrics.go already merges
   ctrlruntimemetrics.Registry into the scrape handler — confirm that.

Acceptance criteria:
- A scrape (or a unit test of pkg/metrics) includes rest_client_request_duration_seconds,
  rest_client_dns_resolution_duration_seconds, rest_client_request_size_bytes,
  rest_client_response_size_bytes, rest_client_rate_limiter_duration_seconds, and
  rest_client_request_retries_total alongside the existing rest_client_requests_total.
- go test ./pkg/metrics/... passes and no duplicate-registration panic occurs with multiple
  Prow binaries importing the package.
- go vet ./pkg/metrics/... is clean.

Scope guard:
- Do not replace or reconfigure Prow's custom metrics, pushgateway behaviour, or the
  ctrlruntimemetrics.Registry unregistration logic.
- Do not add per-component labels or rename existing metrics.
- Stay on controller-runtime v0.25.0.
```
