---
pr: kubernetes-sigs/prow#742
title: "chore(deps): bump github.com/in-toto/in-toto-golang from 0.9.0 to 0.11.0 in /hack/tools"
head_sha: 5aa242d1ccb5be7f7555b1c4bf8bf485504d1963
base: main
reviewed_at: 2026-07-26T23:27:08Z
verdict: approve
---

## What this PR does
- Dependabot bump of `github.com/in-toto/in-toto-golang` from v0.9.0 to v0.11.0 in `hack/tools/go.mod`/`go.sum`.
- Indirect dependency of the `hack/tools` module (build/dev tooling), not the main `prow` module.
- No project code changes; only manifest/lockfile churn (3 additions / 3 deletions across 2 files).

## Findings

(none)

## Checked
- Release freshness: v0.11.0 tagged 2026-05-04 per proxy.golang.org (~83 days old at review time) — well past soak-time threshold.
- Direct usage: `grep -rln '"github.com/in-toto/in-toto-golang' hack --include='*.go'` returns nothing — not imported directly, not vendored.
- Changelog v0.9.0..v0.11.0: mostly routine dependency bumps (grpc, x/sys, go-jose, testify); one functional fix (`in-toto/in-toto-golang#462`, negation-character-class matching) with no bearing on our code since we don't import the module directly.
- Scope: manifest/lockfile-only change in the `hack/tools` module; no vendored code touched.

## Open questions
(none)
