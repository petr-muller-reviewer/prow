---
pr: kubernetes-sigs/prow#966
title: "config: expose loaded job definition counts"
head_sha: 8823ecb279c05df966ffe4e999c18d70a069e9df
base: main
reviewed_at: 2026-09-23T21:39:49Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-09-27T15:22:02Z
  gated_head_sha: 8823ecb279c05df966ffe4e999c18d70a069e9df
  reviewed_head_sha: 8823ecb279c05df966ffe4e999c18d70a069e9df
---

# Review

## Gate

Do not merge. The PR remains at the reviewed head. The blocking test finding and the metric contract findings remain unaddressed.

- **Blocks merge — `REVIEW.md`, “Cover the published metric's values and refresh behavior”:** `pkg/config/agent.go:451-454` still sets all three gauge values, but the PR adds no test for their counts or replacement after another `Set` or `SetWithoutBroadcast`. Add a focused test covering non-uniform counts across repositories and a subsequent refresh.
- **Also gates merge — `REVIEW.md`, “Define the metric's attribution and scope for operators”:** `pkg/config/agent.go:34-40` still exposes only a `type` label, and `site/content/en/docs/metrics/_index.md` does not describe this metric. Document that presubmit and postsubmit counts exclude dynamic in-repo jobs and that component attribution depends on scrape-target labels.
- **Needs intent confirmed — `REVIEW.md`, “Say when the counts are measured”:** `pkg/config/agent.go:413-454` records counts on config load, not on job use. Horologium reports all three types while consuming only periodics (`cmd/horologium/main.go:139-176`). Confirm that loaded-snapshot inventory is intended and state this in the metric contract; otherwise instrument consumption instead.
- **Also gates merge — `REVIEW.md`, “Name and include the in-repo job population”:** `config_job_definitions` at `pkg/config/agent.go:34-38` omits in-repo jobs. They are fetched later for specific refs (`pkg/config/config.go:444-465`, `pkg/config/cache.go:304-337`). Define separate, coherent measurement semantics for them and distinguish static from in-repo definitions in the metric name or labels.

Independent merge risk: The full PR diff is 26 added lines in `pkg/config/agent.go`. It adds a globally registered gauge to processes importing `pkg/config`, with three bounded label values after a config snapshot is set. Its broad name can be mistaken for a complete inventory; its values can also be mistaken for per-component consumption or summed across processes as unique definitions, producing misleading monitoring. It changes no exported Go API, configuration schema, flag, default, job execution path, or wire format; no backward-incompatible deployment risk was found.

## Verdict

Request changes.

The implementation is localized and has low deployment risk, but the published metric lacks regression coverage and a clear scope. It records loaded static definitions in every config-agent process while its name implies a complete job-definition count, including in-repo jobs. Define that contract and test its values and refresh behavior before merging.

## What this PR does

- Registers a `config_job_definitions` gauge vector with a bounded `type` label.
- Emits counts for periodic, static presubmit, and static postsubmit definitions.
- Updates those counts when `Agent.Set` accepts a configuration snapshot.
- Updates those counts on the non-broadcasting configuration update path as well.

## Findings

### [blocking] Cover the published metric's values and refresh behavior
- where: `pkg/config/agent.go:451-454`
- excerpt: |
    func recordJobDefinitionMetrics(c *Config) {
        configJobDefinitions.WithLabelValues("periodic").Set(float64(len(c.Periodics)))
        configJobDefinitions.WithLabelValues("presubmit").Set(float64(jobCount(c.PresubmitsStatic)))
        configJobDefinitions.WithLabelValues("postsubmit").Set(float64(jobCount(c.PostsubmitsStatic)))
    }
- concern: The PR introduces an externally consumed metric but adds no coverage for its three label values or for replacing values after a later configuration update. Add a focused test with non-uniform periodic, presubmit, and postsubmit counts across repositories, and verify a subsequent `Set` or `SetWithoutBroadcast` refresh overwrites the earlier values.

### [should-fix] Define the metric's attribution and scope for operators
- where: `pkg/config/agent.go:34-40`
- excerpt: |
    var configJobDefinitions = prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Name: "config_job_definitions",
        Help: "Number of job definitions in the accepted configuration, by job type.",
    }, []string{"type"})
- concern: The metric distinguishes only job type. Component/source attribution must therefore come from Prometheus scrape-target labels, and presubmit/postsubmit values intentionally count static configuration rather than dynamically resolved in-repo jobs. Document these semantics where Prow metrics are described so alerts and aggregate queries are not misleading.

### [should-fix] Say when the counts are measured
- where: `pkg/config/agent.go:413-448`
- excerpt: |
    ca.c = c
    recordJobDefinitionMetrics(c)
- concern: Both `Set` and `SetWithoutBroadcast` record counts as soon as a config snapshot is accepted, regardless of which jobs the component later consumes. For example, Horologium consumes periodic jobs but also reports the loaded presubmit and postsubmit counts. Confirm that this is intended to measure loaded definitions rather than use, and make that distinction explicit in the metric help and documentation; if use is intended, instrument the consumers.

### [should-fix] Name and include the in-repo job population
- where: `pkg/config/agent.go:34-38`
- excerpt: |
    var configJobDefinitions = prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Name: "config_job_definitions",
        Help: "Number of job definitions in the accepted configuration, by job type.",
    }, []string{"type"})
- concern: This broad name implies complete job-definition counts, while the implementation omits in-repo presubmits and postsubmits. Those definitions are fetched on demand for specific repository refs, so they cannot simply be added to the load-time snapshot gauge. Define how in-repo counts will be exposed with a clear unit and lifecycle, and distinguish static from in-repo definitions in the name or labels.

## Checked

- The local checkout matches PR head `8823ecb279c05df966ffe4e999c18d70a069e9df`.
- The count helper sums all static jobs across repository keys and does not introduce repository-level cardinality.
- Both `Set` and `SetWithoutBroadcast` update the gauge after installing the snapshot.
- No configuration schema, job execution behavior, migration requirement, or permission boundary changes.
- The fixed `type` vocabulary bounds the metric to three series per process.

## Open questions

- Will the deployment scrape configuration retain labels that identify the emitting component? The metric itself has only `type`, so queries must use scrape-target labels to distinguish sources.
- Is the intended signal a count of definitions loaded by each process, or definitions actually consumed by that component?
- How should in-repo definitions fetched for different repository refs be counted without mixing revisions into one gauge?
