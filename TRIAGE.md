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
advice:
  advised_at: 2026-08-21T16:02:50Z
  based_on_triaged_at: 2026-06-29T15:24:32Z
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

## FOLLOWUP: agentic-coding candidate

**Do not reopen the issue** — decision made 2026-08-21, overrides the "reopen" advice below/in history.

This is a good candidate for an experiment: hand the *original issue text* (not PR #573) to a strong coding agent, let it implement optional TLS for Deck and Hook independently, and **compare the result against PR #573** (open, `CHANGES_REQUESTED`, stalled since 2026-06-12 — see `## Advice` below for the full PR history). Scope is small, well-precedented (three reference implementations already in-tree: `cmd/admission/main.go`, `cmd/jenkins-operator/main.go`, `test/integration/cmd/fakegitserver/main.go`), and the existing PR's review feedback (flag-naming/CA-vs-client-cert confusion, missing docs, no cert-rotation story) gives a ready-made rubric to grade the agent's output against.

Comparison angles worth checking: does the agent avoid the `--client-cert-file` naming confusion flagged in PR #573's review; does it cover both Deck and Hook (not just one); is the cert+key pairing validated (required together or neither, per the `jenkins-operator` pattern); does it add tests and docs unprompted; how much back-and-forth review would it need versus the two rounds already spent on #573.

## Advice

No new issue-side activity since the 2026-06-29 refresh (state, labels, comments unchanged). Live-state gathering surfaced a linked PR that predates the refresh but was never folded into `TRIAGE.md`/`TRIAGE.html`: **PR #573** ("Adding option to enable Back End HTTPS for Prow Ingress"), opened 2025-12-10 by the issue's own assignee (NiJuFirenzia), implementing exactly the `--tls-cert-file`/`--tls-private-key-file` pattern this triage anticipated, touching `cmd/deck/main.go`, `cmd/hook/main.go`, and a new `pkg/flagutil/ssl.go`. It carries `area/deck`/`area/hook` labels and is `CHANGES_REQUESTED` (reviews from petr-muller and ivankatliarchuk — flag naming/stuttering nits, a `--client-cert-file` naming issue that's misleading about CA-vs-client-cert semantics, and a request for operator documentation). Last commit 2026-02-25; last activity is the author's 2026-06-12 comment promising to add documentation and address feedback — nothing since (over 2 months).

~~1. Reopen the issue — bot auto-closed it as `not-planned` on 2026-06-28 for `lifecycle/rotten` inactivity, but there is a live, labeled, in-progress PR (#573) implementing it.~~ **Rejected 2026-08-21: keep the issue closed** (see `## FOLLOWUP` above — the agentic-coding experiment is the preferred next step, not reopening).

2. **Remove `lifecycle/rotten`** — rationale: cosmetic cleanup regardless of open/closed state; the label no longer reflects an active-vs-abandoned assessment now there's a concrete follow-up plan.
   ```
   gh issue edit 328 --repo kubernetes-sigs/prow --remove-label lifecycle/rotten
   ```

3. **Add `area/deck`, `area/hook`** — rationale: matches the labels already applied to PR #573; keeps the issue discoverable/filterable even while closed. Skip `help-wanted`/`good-first-issue`: the issue is already self-assigned and has an active PR, so soliciting outside contributors would duplicate effort.
   ```
   gh issue edit 328 --repo kubernetes-sigs/prow --add-label area/deck,area/hook
   ```

4. **Ping PR #573, don't duplicate effort on the human side** — rationale: the PR directly resolves this issue and is under active review, not abandoned, but has been stalled for over two months since the author's last promise to address feedback (naming fix, docs) and over five months since the last commit.
   ```
   gh pr comment 573 --repo kubernetes-sigs/prow --body "Checking in — it's been a couple of months since your last update here (2026-06-12) where you mentioned you'd add documentation and address the flag-naming/stuttering feedback. Is this still something you're planning to pick up? Happy to review again once it's updated."
   ```

## Open questions

- Should Deck and Hook TLS be in a single PR or separate?
- Should the PR also update example Kubernetes deployment manifests in `config/prow/`?
- Does the author's prototype account for Hook receiving webhooks over TLS — does this interact with GitHub's webhook delivery in any meaningful way?
