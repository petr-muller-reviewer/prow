---
issue: kubernetes-sigs/prow#328
title: "Allow Option for Ingress to Reach pods through SSL"
state: closed
labels: kind/feature, lifecycle/rotten
main_sha: 71428b9c282ee8c9e7e9512068fccce86e7915da
triaged_at: 2026-06-29T15:24:32Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/feature, area/deck, area/hook, help-wanted]
refresh_log:
  - at: 2026-05-30T13:20:42Z
    summary: initial triage
  - at: 2026-06-29T15:24:32Z
    summary: k8s-triage-robot closed issue as "Not Planned" on 2026-06-28 after lifecycle/rotten inactivity; no author or maintainer activity since initial triage
---

## Findings

### [cause] Deck and Hook have no TLS server option
- detail: Both components call only `interrupts.ListenAndServe()` (HTTP). `interrupts.ListenAndServeTLS()` exists in the package but is not wired up to either component.
- evidence: `cmd/deck/main.go:505-514`, `cmd/hook/main.go:263-277`

### [related-code] interrupts.ListenAndServeTLS already implemented
- where: `pkg/interrupts/interrupts.go:171-181`
- excerpt: |
    func ListenAndServeTLS(server *http.Server, certFile, keyFile string, gracePeriod time.Duration)
    // calls server.ListenAndServeTLS(certFile, keyFile) with graceful shutdown

### [related-code] Deck server startup — HTTP only
- where: `cmd/deck/main.go:505-514`
- excerpt: |
    server := &http.Server{Addr: ":8080", Handler: ...}
    interrupts.ListenAndServe(server, 5*time.Second)

### [related-code] Hook server startup — HTTP only
- where: `cmd/hook/main.go:263-277`
- excerpt: |
    httpServer := &http.Server{Addr: o.port, Handler: ...}
    interrupts.ListenAndServe(httpServer, o.gracePeriod)

### [related-code] Reference implementation: admission with required TLS flags
- where: `cmd/admission/main.go:52-59,85`
- excerpt: |
    fs.StringVar(&o.tlsCertFile, "tls-cert-file", "", "...")
    fs.StringVar(&o.tlsPrivateKeyFile, "tls-private-key-file", "", "...")
    // ...
    interrupts.ListenAndServeTLS(server, o.tlsCertFile, o.tlsPrivateKeyFile, gracePeriod)

### [related-code] Reference implementation: jenkins-operator with optional TLS
- where: `cmd/jenkins-operator/main.go`
- excerpt: |
    // --cert-file, --key-file, --ca-cert-file; all three or none

### [related-code] Reference implementation: fakegitserver with conditional TLS
- where: `test/integration/cmd/fakegitserver/main.go`
- excerpt: |
    if cert != "" && key != "" {
        interrupts.ListenAndServeTLS(server, cert, key, gracePeriod)
    } else {
        interrupts.ListenAndServe(server, gracePeriod)
    }

### Since previous triage (2026-05-30 → 2026-06-29)

- No author or contributor activity since initial triage.
- 2026-06-28: k8s-triage-robot issued `/close not-planned` per lifecycle/rotten policy (30d inactivity after rotten label); issue closed automatically.
- No linked PRs appeared; author's promised prototype never materialized as a PR.

## Checked

- `interrupts` package already has `ListenAndServeTLS` — no changes needed there
- Three reference implementations in the same repo (admission, jenkins-operator, fakegitserver)
- `lifecycle/rotten` is stale-bot automation, not a maintainer close decision; author reopened Oct 2025 and committed to a PR
- TLS on Deck's listening side is independent of Deck's backend HTTP calls (to GCS etc.) — no interference
- Adding optional flags is fully backwards compatible; HTTP-only path unchanged

## Next steps

- **Reopen** the issue (`/reopen`) — the bot closed a legitimate accepted feature request; no maintainer made a close decision.
- Remove `lifecycle/rotten` label (`/remove-lifecycle rotten`)
- Apply `/area deck`, `/area hook`, `/help-wanted`
- Wait for author's PR (they claimed a working prototype using `interrupts.ListenAndServeTLS()`)
- When PR arrives, verify: both Deck and Hook covered; TLS is opt-in (HTTP default preserved); both cert+key flags required together or neither; options validation tested; consider whether example Kubernetes manifests need updating

## Open questions

- Should Deck and Hook TLS be in a single PR or separate?
- Should the PR also update example Kubernetes deployment manifests in `config/prow/`?
- Does the author's prototype account for Hook receiving webhooks over TLS — does this interact with GitHub's webhook delivery in any meaningful way?
