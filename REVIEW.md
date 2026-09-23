---
pr: kubernetes-sigs/prow#966
title: "config: expose loaded job definition counts"
head_sha: 8823ecb279c05df966ffe4e999c18d70a069e9df
base: main
reviewed_at: 2026-09-23T21:39:49Z
verdict: request-changes
---

# Review

## Verdict

Request changes.

The implementation is localized, correct for the intended static configuration counts, and has low deployment risk. However, it adds a public observability contract without regression coverage; a focused test is needed to preserve the three-series and refresh semantics during future config-agent changes.

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

## Checked

- The local checkout matches PR head `8823ecb279c05df966ffe4e999c18d70a069e9df`.
- The count helper sums all static jobs across repository keys and does not introduce repository-level cardinality.
- Both `Set` and `SetWithoutBroadcast` update the gauge after installing the snapshot.
- No configuration schema, job execution behavior, migration requirement, or permission boundary changes.
- The fixed `type` vocabulary bounds the metric to three series per process.

## Open questions

- Will the deployment scrape configuration retain labels that identify the emitting component? The metric itself has only `type`, so queries must use scrape-target labels to distinguish sources.
