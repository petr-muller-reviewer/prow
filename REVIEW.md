---
pr: kubernetes-sigs/prow#742
title: "chore(deps): bump github.com/in-toto/in-toto-golang from 0.9.0 to 0.11.0 in /hack/tools"
head_sha: 5aa242d1ccb5be7f7555b1c4bf8bf485504d1963
base: main
reviewed_at: 2026-08-08T22:57:21Z
verdict: approve
---

## What this PR does
- Dependabot bump of `github.com/in-toto/in-toto-golang` from v0.9.0 to v0.11.0 in `hack/tools/go.mod`/`go.sum`.
- Indirect dependency of the `hack/tools` module (build/dev tooling), not the main `prow` module.
- No project code changes; only manifest/lockfile churn (3 additions / 3 deletions across 2 files).

## Findings

(none)

## Checked
- Release freshness: v0.11.0 tagged/released 2026-05-04T16:21:36Z per proxy.golang.org and GitHub release — ~96 days old at this review, real tagged release (not a pseudo-version), published by core maintainer `adityasaky`. Well past soak-time threshold.
- Direct usage: `grep -rln '"github.com/in-toto/in-toto-golang' --include='*.go' .` (excluding vendor) returns nothing — not imported anywhere in project Go source.
- Changelog v0.9.0..v0.11.0: overwhelmingly routine transitive dependency bumps (grpc, x/sys, x/net, x/crypto, go-jose, testify, cobra, go-cmp, GH Actions versions). Substantive upstream changes: "Add match products feature" (#237), "Drop use of `any` for hash objects" (#238), "Deprecate Provenance v1 struct in favor of /attestation protobufs" (#267), "Fixes filepath pattern matching in windows" (#254), and in v0.11.0 specifically "match: Replace `^` with `!` for negation in character classes" (#462, in-toto artifact-rule matching bugfix). No CVEs/security advisories in either release's notes.
- Exposure: since the module isn't imported by any project code (pulled in only transitively, in a build-tooling-only module that isn't shipped), none of the upstream changes — including the negation-matching fix — touch anything actually exercised. Exposure is effectively zero.
- Scope: manifest/lockfile-only change in the `hack/tools` module; no vendored code touched.

## Open questions
(none)
