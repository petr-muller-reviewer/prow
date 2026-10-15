---
pr: kubernetes-sigs/prow#966
title: "config: expose loaded job definition counts"
head_sha: 49a050829080b79ce689eecfa225affd142b1085
base: main
reviewed_at: 2026-10-15T23:14:10Z
verdict: approve
reassessed_at: 2026-10-10T18:19:59Z
refresh_log:
  - at: 2026-10-09T12:46:33Z
    old_sha: 8823ecb279c05df966ffe4e999c18d70a069e9df
    new_sha: 49a050829080b79ce689eecfa225affd142b1085
    summary: "Reviewed targeted collector rewrite and component registration; resolved scope and attribution documentation, retained missing coverage."
  - at: 2026-10-15T23:14:10Z
    old_sha: 49a050829080b79ce689eecfa225affd142b1085
    new_sha: 49a050829080b79ce689eecfa225affd142b1085
    summary: "No code changes; incorporated maintainer approval and registration discussion, author acknowledgment, and merge on October 10. Retained the local approve reassessment and separate followup."
gate:
  decision: merge
  gated_at: 2026-10-15T23:17:39Z
  gated_head_sha: 49a050829080b79ce689eecfa225affd142b1085
  reviewed_head_sha: 49a050829080b79ce689eecfa225affd142b1085
---

# Review

## Gate

Merge. PR #966 already merged on 2026-10-10 at 18:45:16 UTC as `b8bd4f20ea0bb4c10da7da66bf84b8d70194bcc0`; its head remains the reviewed `49a050829080b79ce689eecfa225affd142b1085`, with no subsequent code delta. Prior substantive feedback is addressed or explicitly dispositioned as nonblocking, and the independent compatibility pass identified no unacceptable risk.

- **Not addressed, accepted as nonblocking — REVIEW.md, “Cover the published metric's values and refresh behavior” (`pkg/config/agent.go:69-81`):** The PR does not add focused collector tests. The saved reassessment accepts this as a test suggestion because no incorrect count or refresh behavior was identified; it does not gate merge.
- **Addressed by discussion, nonblocking — @petr-muller, collector publication (`cmd/crier/main.go:322`, `cmd/exporter/main.go:109-116`):** The maintainer acknowledged the existing registration conventions on October 10 and subsequently approved. Exporter's registrations cover both its custom scrape registry and default Pushgateway registry. Centralization remains a separate accepted followup.

Gating list: none. Naming, static scope, source attribution, and agent ownership findings are addressed. No unresolved substantive GitHub feedback requires a change or confirmation.

Independent merge risk: The full PR adds 93 lines and removes 4 across 15 files, including registration in 13 component entry points. Each registered agent emits three bounded series from a coherent cached snapshot. Describe and Collect add methods without removing or changing an existing exported signature; configuration schemas, flags, defaults, and job execution behavior are unchanged. Deployment impact is an additive metric with no migration or coordinated rollout. Documentation explains static scope and replica attribution. No notable merge risk was found. This gate used direct code inspection, with no applicable repository compatibility skill; it did not rerun tests. Prior local test results remain recorded in Checked.

## Verdict

Approve, with a nonblocking test suggestion.

The implementation records coherent static job counts when each config agent accepts a snapshot and exposes them through established publication paths. Naming and documentation describe the scope and attribution. Missing focused tests are a useful improvement, but do not justify requesting changes without an identified correctness defect. The separate local metrics refactor is not required for this PR to merge.

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

Since previous review (2026-10-15 refresh; no code changes):

- On 2026-10-10, @petr-muller raised collector-publication centralization at 18:09:47 UTC, acknowledged existing registration patterns at 18:18:06 UTC, indicated acceptance at 18:22:50 UTC, and approved at 18:23:14 UTC. This supports the saved disposition of centralization as a separate followup.
- PR #966 merged at 2026-10-10T18:45:16Z into commit `b8bd4f20ea0bb4c10da7da66bf84b8d70194bcc0`, with reviewed head unchanged at `49a050829080b79ce689eecfa225affd142b1085`.
- @Prucek acknowledged the review on 2026-10-12 at 07:28:44 UTC. The local approve reassessment, nonblocking coverage suggestion, and accepted followup are preserved.

## Findings

### [should-fix] Cover the published metric's values and refresh behavior
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
- concern: Nonblocking test suggestion. The collector adds no focused tests for this metric, although inspection found no incorrect counts or refresh behavior. Consider registry-based coverage for startup absence, unequal counts across repositories, replacement including zeros through both Set and SetWithoutBroadcast, and independent agents.

## Resolved

### [should-fix] Integrate collector publication in one place
- where: `cmd/exporter/main.go:109-116`
- excerpt: |
    registry.MustRegister(configAgent)
    // Pushgateway collection uses the default registry.
    prometheus.MustRegister(configAgent)
    registry.MustRegister(prowjobs.NewProwJobLifecycleHistogramVec(informerFactory.Prow().V1().ProwJobs().Informer()))
    metrics.ExposeMetricsWithRegistry("exporter", cfg().PushGateway, o.instrumentationOptions.MetricsPort, registry, nil)
- concern: Withdrawn after inspecting existing command registration and output architecture. MustRegister followed by ExposeMetrics is an established, valid integration pattern. Exporter correctly registers the collector in both output registries; no omission was found. A shared gatherer and centralized command registration are broader architectural improvements, not requirements for this metric PR.

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
- concern: Dispositioned by the design discussion: the objective is observing configured static definitions a component sees. Recording the accepted snapshot is the appropriate measurement point, including for types the component does not consume. The current name and documentation clarify this scope.

### [should-fix] Name and include the in-repo job population
- where: `pkg/config/agent.go:34-38`
- excerpt: |
    var configJobDefinitions = prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Name: "config_job_definitions",
        Help: "Number of job definitions in the accepted configuration, by job type.",
    }, []string{"type"})
- concern: Withdrawn as a scope expansion: this metric can be complete for static inventory without including dynamically fetched in-repo definitions. Their revision-dependent cache population is a separate measurement. Static scope has been clarified; additional in-repo instrumentation is not a merge requirement.

## Checked

- The refreshed head was inspected directly from git objects for `49a050829080b79ce689eecfa225affd142b1085`; the working checkout was not reset or updated during that refresh. Subsequently, local collector tests were added and passed against this PR implementation.
- The count helper sums all static jobs across repository keys and does not introduce repository-level cardinality.
- Both Set and SetWithoutBroadcast cache counts under the agent lock after installing the snapshot. Collect copies all counts and the loaded state under a read lock, then emits metrics after releasing the lock.
- No configuration schema, job execution behavior, migration requirement, or permission boundary changes.
- The fixed `type` vocabulary bounds the metric to three series per process.
- Package-level collectors registered in init are established conventions in pkg/config/cache.go, pkg/config/inrepoconfig.go, pkg/kube/metrics.go, and pkg/tide/tide.go. This instrumentation style is not itself a finding.
- pkg/metrics/prowjobs provides informer-driven lifecycle histograms; cmd/exporter contains a custom scrape-time collector. These are alternative patterns, not required architecture for this PR.
- No requirement to count dynamically fetched in-repo definitions or instrument consumer execution for the static inventory objective.
- No samples are emitted before the first snapshot; an accepted empty config emits three zeros.
- Collector state belongs to each Agent, removing cross-agent overwrites. Distinct agents would still require distinct registries or distinguishing registration labels if exported together.
- PR is MERGED as of 2026-10-10T18:45:16Z; the head is unchanged from the reviewed commit. Since the prior review baseline, maintainer discussion and approval confirm registration centralization is nonblocking; the author subsequently acknowledged the review.

## Open questions

- None that block this PR. The shared-gatherer improvement is recorded below as an accepted followup.

## Followups

Followup walkthrough on 2026-10-15: retained one previously accepted architectural followup; skipped the additional collector-regression-test candidate at the maintainer's direction. No new work was accepted. The nonblocking test suggestion remains in Findings for review history, not as an accepted handoff.

### Centralize command collector registration and share one gatherer

- Category: architecture
- Necessity: could — simplifies publication and prevents future divergence between scrape and push outputs.
- Status: accepted by the maintainer; implemented in separate local commits, not published.
- Where: `pkg/metrics/metrics.go`, `pkg/metrics/metrics_test.go`, command entry points under `cmd/`, and `site/content/en/docs/metrics/_index.md`.
- Why followup: PR #966 correctly registers its collector for the existing output paths. The split between exporter's custom scrape registry and the default Pushgateway registry predates this PR; changing it affects existing metrics and belongs in a separate change.
- Local implementation: `92d519cd0` on `metrics-expose-collectors` centralizes command-owned collectors, including existing Deck, Gerrit, Moonraker, Sinker, and ghproxy metrics. `6c3e28edd` on `metrics-unified-gatherer` builds on it to use the same gatherer for both outputs. Exposure tests and compilation of affected commands passed; the metrics package tests passed after replaying the second commit.

```text
In kubernetes-sigs/prow, following merged PR #966 (config: expose loaded job definition counts), merge commit b8bd4f20ea0bb4c10da7da66bf84b8d70194bcc0, centralize command-owned collector registration and make HTTP scraping and Pushgateway pushing use one configured gatherer. Work from the merged default branch in the task's own worktree.

Start by inspecting the separate local implementation, if available: 92d519cd0 (Centralize command-owned metric exposure) and 6c3e28edd (Use one gatherer for scraped and pushed metrics), on metrics-expose-collectors and metrics-unified-gatherer respectively. Reuse and adapt these commits to the current upstream code rather than rebuilding the change without inspecting them.

In pkg/metrics/metrics.go, register additional command collectors once, combine the selected registry (default or explicitly supplied) with controller-runtime and those collectors, and send that same gatherer to both HTTP and Pushgateway outputs. Normalize duplicate Go/process collectors before either output starts, including push-only mode. Preserve endpoint, interval, port, and serve_metrics configuration behavior.

Update config-agent registration across the affected command entry points and fold in the existing command-owned collectors in Deck, Gerrit, Moonraker, Sinker, and ghproxy. Remove exporter's duplicate registration of the config agent while retaining its explicit custom registry. Keep library-owned collector registration outside this migration.

Acceptance criteria: tests in pkg/metrics/metrics_test.go verify default and custom registry behavior, additional collectors, controller-runtime collectors, and push-only and combined HTTP/push modes; both outputs receive the same metric families; HTTP gathering succeeds without duplicate runtime families. Compile all changed commands. Document that exporter custom metrics and controller-runtime metrics now participate in pushing, since this changes the pushed metric population.

Scope: a separate architectural improvement, not a prerequisite for merging PR #966. Do not change job-definition counting semantics, add in-repo job counts, remove Pushgateway support, or publish anything without explicit authorization.
```
