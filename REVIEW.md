---
pr: kubernetes-sigs/prow#628
title: "fix(status-reconciler): fail closed on config load errors and add operational metrics"
head_sha: 37e1c4b136b68438750c942fe5113adf3675e0f0
base: main
reviewed_at: 2026-10-05T22:28:17Z
verdict: request-changes
---
# Review

## Verdict

Request changes. Three maintainer perspectives converged on two confirmed gaps: config mtime errors do not mark status-reconciler unhealthy, and the new health registration model leaves existing callers without `/healthz`. Startup probe timing may also restart a slow-starting pod before its health route is installed.

## What this PR does

- Makes corrupted saved status fail to load instead of continuing from empty state.
- Propagates config-agent load errors to status-reconciler health state.
- Adds liveness and readiness probes to the starter deployment variants.
- Adds metrics for loaded presubmits and retired contexts.
- Refactors config-agent options and updates status-reconciler and deck call sites.

## Findings

### [blocking] Route config stat failures through the health callback
- where: `pkg/config/agent.go:371-373`
- concern: If the config path cannot be statted, the poller skips the reload without calling `onError`. Status-reconciler can therefore continue reporting healthy while config updates are unavailable; mark it unhealthy on this path and cover it with a focused test.
- excerpt: |
    recentModTime, err := lastConfigModTime(prowConfig, jobConfig)
    if err != nil {
        continue
    }

### [blocking] Complete the `/healthz` registration migration
- where: `pkg/pjutil/health.go:46-49`
- concern: The constructor now starts the server with an empty mux, but five integration fake servers still call `NewHealth()` followed only by `ServeReady()`. Their `/healthz` endpoint returns 404; preserve the constructor's prior default route or register `ServeLive()` in all remaining callers.
- excerpt: |
    func NewHealthOnPort(port int) *Health {
        healthMux := http.NewServeMux()
        server := &http.Server{Addr: ":" + strconv.Itoa(port), Handler: healthMux}
        interrupts.ListenAndServe(server, 5*time.Second)
        return &Health{
            healthMux: healthMux,
        }
    }

### [question] Could startup probes allow time for health registration?
- where: `cmd/status-reconciler/main.go:147-148`
- concern: `/healthz` is registered after client and storage initialization, while the starter liveness probe begins after three seconds. If initialization exceeds the probe failure window, Kubernetes may restart the pod before the handler is available. Consider registering a startup-safe liveness handler earlier or adding a startup probe / longer startup allowance.
- excerpt: |
    c := statusreconciler.NewController(o.continueOnError, o.getDenyList(), o.getDenyListAll(), opener, o.config, o.statusURI, prowJobClient, githubClient, pluginAgent)
    health.ServeLive(c.Healthy)
    interrupts.Run(func(ctx context.Context) {
        c.Run(ctx)
    })

## Checked

- The controller health flag uses atomic accessors, and the config-agent `Load` error path invokes the callback.
- Starter manifests contain readiness and liveness probes for status-reconciler.
- Functional config-agent options make reuse, error handling, and additional config transformations explicit at call sites.
- No tests were run as part of this review.

## Open questions

- Can the liveness handler be registered before initialization, or should the starter manifests use a startup probe to avoid restarts during slow startup?
