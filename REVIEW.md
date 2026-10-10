---
pr: kubernetes-sigs/prow#966
title: "config: expose loaded job definition counts"
head_sha: 49a050829080b79ce689eecfa225affd142b1085
base: main
reviewed_at: 2026-10-09T12:46:33Z
verdict: request-changes
refresh_log:
  - at: 2026-10-09T12:46:33Z
    old_sha: 8823ecb279c05df966ffe4e999c18d70a069e9df
    new_sha: 49a050829080b79ce689eecfa225affd142b1085
    summary: "Reviewed targeted collector rewrite and component registration; resolved scope and attribution documentation, retained missing coverage."
gate:
  decision: do-not-merge
  gated_at: 2026-10-10T17:53:52Z
  gated_head_sha: 49a050829080b79ce689eecfa225affd142b1085
  reviewed_head_sha: 49a050829080b79ce689eecfa225affd142b1085
---

# Review

## Gate

Do not merge. Current head `49a050829080b79ce689eecfa225affd142b1085` matches the refreshed review. The blocking test finding remains, and the new publication wiring should be made coherent within this PR. Naming, attribution documentation, and agent ownership are addressed.

- **Not addressed; blocks merge — REVIEW.md, “Cover the published metric's values and refresh behavior” (`pkg/config/agent.go:69-81`):** Collect emits three cached values after a configuration is loaded, but no tests verify the counts, startup absence, zero-count replacement, or independent agent state. Add focused registry-based coverage through Set and SetWithoutBroadcast to unblock the gate.
- **Not addressed; also gates merge — REVIEW.md, “Integrate collector publication in one place” (`cmd/exporter/main.go:109-116`, `pkg/metrics/metrics.go:38-88`):** This PR adds registration beside `ExposeMetrics` in 12 standard components and registers exporter in both a custom and the default registry because serving and pushing use different gatherers. Make the publication path explicit in the metrics API or another single integration point, remove the new double-registration workaround, and verify both scrape and Pushgateway paths before merging.

Prior GitHub feedback from @petr-muller on 2026-10-05 requested the name prow_config_static_job_definition and raised singleton/attribution concerns. Current code adopts that name, implements an agent-owned collector, and documents scrape-target attribution; these items are addressed. No other substantive GitHub feedback is unresolved.

Independent merge risk: The full PR changes 15 files with 93 additions and 4 deletions, adding an explicitly registered collector to 13 component entry points. Describe and Collect are additive exported methods; no existing exported signature, configuration schema, flag, default, job execution path, or wire format is removed or changed. The metric contributes three series per registered agent. Existing deployments need no migration or coordinated rollout. The new publication wiring is a maintainability and coverage risk: the custom scrape and default Pushgateway registries can silently diverge if a collector is registered in only one. Documentation covers static scope and avoiding sums across replicas. The assessment used direct code inspection; no applicable repository compatibility skill was available, and no test execution at this head is claimed.

## Verdict

Request changes.

The updated implementation binds cached static counts to each config agent. The metric name and documentation explain static snapshot scope, replica attribution through scrape-target labels, and the exclusion of in-repo jobs, resolving the contract finding. Request changes remains for missing regression coverage and for publication wiring introduced by this PR: the exporter must currently register one collector twice because its serving and pushing paths use different registries.

## What this PR does

- Exposes `prow_config_static_job_definition` through an explicitly registered config-agent collector with a bounded `type` label.
- Emits counts for periodic, static presubmit, and static postsubmit definitions.
- Caches counts under the agent lock when either configuration update path accepts a snapshot; scrapes read one coherent counts snapshot.
- Registers the collector in 13 component entry points, including exporter's serving and Pushgateway registries.
- Documents static scope and component/instance attribution in the metrics catalog.

Since previous review:

- Prucek replaced the PR commit on 2026-10-08 at 13:39:15 UTC; the old SHA is not an ancestor of the current head. The targeted delta is 81 additions and 18 deletions across 15 existing files, suitable for an in-place refresh.
- Replaced the global gauge with an agent-owned collector, renamed the metric, and added explicit registration in each component.
- Added 13 lines of metrics documentation; no tests or substantive new reviewer comments were added.

## Findings

### [blocking] Cover the published metric's values and refresh behavior
- where: `pkg/config/agent.go:69-81`
- excerpt: |
    ca.mut.RLock()
    counts := ca.jobDefinitions
    loaded := ca.c != nil
    ca.mut.RUnlock()
    if !loaded {
        return
    }
    ch <- prometheus.MustNewConstMetric(configStaticJobDefinitions, prometheus.GaugeValue, float64(counts.periodics), "periodic")
    ch <- prometheus.MustNewConstMetric(configStaticJobDefinitions, prometheus.GaugeValue, float64(counts.presubmits), "presubmit")
    ch <- prometheus.MustNewConstMetric(configStaticJobDefinitions, prometheus.GaugeValue, float64(counts.postsubmits), "postsubmit")
- concern: The collector rewrite still adds no tests for this public metric. Register an agent in a fresh registry and verify no samples before loading, unequal counts across multiple repositories after Set, and replacement with zero counts after SetWithoutBroadcast. Verify separate agents in separate registries retain independent values.

### [should-fix] Integrate collector publication in one place
- where: `cmd/exporter/main.go:109-116`
- excerpt: |
    registry.MustRegister(configAgent)
    // Pushgateway collection uses the default registry.
    prometheus.MustRegister(configAgent)
    registry.MustRegister(prowjobs.NewProwJobLifecycleHistogramVec(informerFactory.Prow().V1().ProwJobs().Informer()))
    metrics.ExposeMetricsWithRegistry("exporter", cfg().PushGateway, o.instrumentationOptions.MetricsPort, registry, nil)
- concern: The PR adds registration beside `ExposeMetrics` in 12 other components and must register exporter twice because `ExposeMetricsWithRegistry` serves the supplied registry but pushes the default one. This makes a new metric depend on two manually synchronized publication paths; missing one registration would silently omit it from one destination. Choose a single integration point for this collector within this PR, with explicit scrape and Pushgateway behavior, and test both paths rather than leaving the double-registration workaround for later cleanup.

## Resolved

### [should-fix] Define the metric's attribution and scope for operators
- where: `pkg/config/agent.go:34-40`
- excerpt: |
    var configJobDefinitions = prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Name: "config_job_definitions",
        Help: "Number of job definitions in the accepted configuration, by job type.",
    }, []string{"type"})
- concern: Resolved in 49a050829080b79ce689eecfa225affd142b1085. The metric is now named prow_config_static_job_definition, its help explicitly says static definitions in accepted configuration, and site/content/en/docs/metrics/_index.md:66-76 documents the job types, exclusion of in-repo jobs, zero counts, attribution through scrape-target labels, and duplicate counts across replicas. The excerpt above preserves the original reviewed implementation.

### [should-fix] Say when the counts are measured
- where: `pkg/config/agent.go:413-448`
- excerpt: |
    ca.c = c
    recordJobDefinitionMetrics(c)
- concern: Dispositioned by the design discussion: the objective is observing configured static definitions a component sees. Recording the accepted snapshot is the appropriate measurement point, including for types the component does not consume. The documentation requirement is retained in the active scope finding.

### [should-fix] Name and include the in-repo job population
- where: `pkg/config/agent.go:34-38`
- excerpt: |
    var configJobDefinitions = prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Name: "config_job_definitions",
        Help: "Number of job definitions in the accepted configuration, by job type.",
    }, []string{"type"})
- concern: Withdrawn as a scope expansion: this metric can be complete for static inventory without including dynamically fetched in-repo definitions. Their revision-dependent cache population is a separate measurement. Clarifying static scope remains an active finding; additional in-repo instrumentation is not a merge requirement.

## Checked

- The refreshed head was inspected directly from git objects for `49a050829080b79ce689eecfa225affd142b1085`; the working checkout was not reset or updated. No test execution at the new head is claimed.
- The count helper sums all static jobs across repository keys and does not introduce repository-level cardinality.
- Both Set and SetWithoutBroadcast cache counts under the agent lock after installing the snapshot. Collect copies all counts and the loaded state under a read lock, then emits metrics after releasing the lock.
- No configuration schema, job execution behavior, migration requirement, or permission boundary changes.
- The fixed `type` vocabulary bounds the metric to three series per process.
- Package-level collectors registered in init are established conventions in pkg/config/cache.go, pkg/config/inrepoconfig.go, pkg/kube/metrics.go, and pkg/tide/tide.go. This instrumentation style is not itself a finding.
- pkg/metrics/prowjobs provides informer-driven lifecycle histograms; cmd/exporter contains a custom scrape-time collector. These are alternative patterns, not required architecture for this PR.
- No requirement to count dynamically fetched in-repo definitions or instrument consumer execution for the static inventory objective.
- No samples are emitted before the first snapshot; an accepted empty config emits three zeros.
- Collector state belongs to each Agent, removing cross-agent overwrites. Distinct agents would still require distinct registries or distinguishing registration labels if exported together.
- PR remains OPEN. No new inline comments or submitted reviews; the only new issue comment is an automated approval-notifier message on 2026-10-08 at 13:39:56 UTC.

## Open questions

- Can the metrics publication API accept the agent collector and make its scrape and Pushgateway registries explicit, so exporter does not need two manual registrations?
