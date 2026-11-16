---
pr: kubernetes-sigs/prow#1070
title: "chore(deps): bump the golang-x group across 1 directory with 3 updates"
head_sha: aa78a5d3d1bf149fb2a69022fcc685b83cceab12
base: main
reviewed_at: 2026-11-16T23:15:26Z
verdict: approve
---

# Review

## Verdict

Approve. Dep-only bump of three directly used `golang.org/x` modules, with no advisory open or fixed in the vulnerability databases, and no changed code on a path Prow actually exercises. The one change worth caring about is a CVE fix in `golang.org/x/text`, and taking it is the right call.

## What this PR does

- Bumps `golang.org/x/oauth2` v0.37.0 → v0.38.0, `golang.org/x/sync` v0.24.0 → v0.25.0, and `golang.org/x/text` v0.43.0 → v0.44.0 in the root module (`go.mod:66`), with matching `go.sum` hashes.
- Dragging `golang.org/x/tools` v0.51.0 → v0.52.0 along indirectly, because x/text v0.44.0's own `go.mod` requires it.
- Mirrors all four pins in `hack/tools/go.mod`, where they are indirect-only.
- No project source, config, or test changes; nothing to line-review.

## Findings

None.

## Resolved

None.

## Checked

- Classification: dep-only. Only `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum` change.
- `golang.org/x/oauth2` v0.38.0: released 2026-11-06 (10 days old), tagged release. Sole commit fixes a `time.NewTicker` panic in `DeviceAccessToken` when the authorization server returns a negative or overflow-inducing `interval` (golang/go#82658). Prow never calls `DeviceAuth`/`DeviceAccessToken` — it uses the `oauth2/google` credentials helpers and token sources — so the changed code is unreachable here.
- `golang.org/x/sync` v0.25.0: released 2026-10-15 (32 days old). Sole commit renames `syncmap.DeepCopyMap` to `CopyMap`; Prow does not import `syncmap`. The `errgroup` and `semaphore` APIs used across `pkg/tide`, `pkg/ghcache`, `pkg/pod-utils/gcs`, `pkg/sidecar`, `pkg/pubsub/subscriber`, and `pkg/bugzilla` are untouched.
- `golang.org/x/text` v0.44.0: released 2026-11-09 (7 days old). Fixes CVE-2026-97044 (unbounded quadratic Punycode encoding, golang/go#82374) and an `Accept-Language` O(N²) guard bypass where `_` skipped the `-` separator count (golang/go#79684). Prow imports only `golang.org/x/text/cases` and `golang.org/x/text/language` in `pkg/plugins/golint/suggestion/golint_suggestion.go:26` and `pkg/spyglass/lenses/links/links.go:31`, calling `cases.Title(language.English)`; it imports neither `idna` nor calls `ParseAcceptLanguage`, so the fixed paths are protected only transitively.
- `golang.org/x/tools` v0.52.0: released 2026-10-10 (37 days old), indirect; reaches only `pkg/plugins/golint` through `golang.org/x/lint` → `astutil`.
- OSV: no advisories for any of the six old/new versions. `CVE-2026-97044` is not yet indexed by OSV, so the database understates the x/text bump; the fixing commit documents it.
- `govulncheck -scan module` at base `86b2c49` and at head: identical, pre-existing findings `GO-2023-1901` (tektoncd/pipeline) and `GO-2026-5932` (x/crypto/openpgp), both in modules this PR does not touch. No advisory in any bumped module, so nothing fixed and nothing left open at the DB level. Symbol-level scan skipped deliberately: the only module-level hits are outside the bump, so a trace would not inform this review. Module scans were run from `cmd/admission` because the repo root has no Go package.
- Build and manifest consistency: `go build ./...` passes at the PR head; `go mod tidy -diff` is clean in both the root module and `hack/tools`.

## Open questions

None.

## Dependency followups

No improvement opportunities identified.

- Modules and ranges examined: `golang.org/x/oauth2` v0.37.0 → v0.38.0, `golang.org/x/sync` v0.24.0 → v0.25.0, `golang.org/x/text` v0.43.0 → v0.44.0, and the transitive `golang.org/x/tools` v0.51.0 → v0.52.0.
- Reason: every changelog delta touches APIs Prow does not call. oauth2's fix is in `DeviceAccessToken` (we never use the device flow); x/sync's only commit renames `syncmap.DeepCopyMap` (we import only `errgroup` and `semaphore`); x/text's fixes are in `idna` and `language.ParseAcceptLanguage` (we import only `cases` and `language.English`); x/tools' 36 commits are gopls/analysis internals (we reach only `astutil` through `golang.org/x/lint`). No deprecations, replacement APIs, or behavior changes at our call sites.
