---
pr: kubernetes-sigs/prow#920
title: "chore(deps): bump google.golang.org/api from 0.294.0 to 0.296.0"
head_sha: e2a802070c54ee319041fe0262b21482b351315e
base: main
reviewed_at: 2026-09-07T15:10:26Z
verdict: approve
---

## Verdict

Approve. This dep-only update moves the direct `google.golang.org/api` requirement from `v0.294.0` to the tagged `v0.296.0` release. The intervening changes are discovery-client regenerations; no Prow-used core API packages changed and no security advisory was published for the upstream module.

## What this PR does

- Updates the direct `google.golang.org/api` requirement in `go.mod` from `v0.294.0` to `v0.296.0`.
- Replaces the corresponding two module checksums in `go.sum`.
- Does not modify Prow source, tests, configuration, or generated project artifacts.

## Findings

None.

## Checked

- Classification: dep-only; only `go.mod` and `go.sum` differ from `main`.
- Release provenance: `v0.296.0` is a real tag from `googleapis/google-api-go-client`, published 2026-08-31 (seven days of soak time at review).
- Upstream range `v0.294.0...v0.296.0`: seven commits, all automated discovery-client regeneration and release commits; no core `googleapi`, `option`, or `iterator` code changed.
- Usage: direct import in five Prow packages. Production use is GCS I/O, Cloud Build, and Secret Manager helpers; tests also use Pub/Sub helpers. The importing paths use `googleapi`, `option`, and `iterator`, not regenerated discovery clients. Only Secret Manager discovery metadata overlaps, while Prow uses the separate Cloud Secret Manager client.
- Security: no published upstream GitHub security advisories returned for this module.
- `git diff --check ffc57790ba3e94ab2081a4a5d498d8baba5db8f1..HEAD` passed.

## Open questions

None.

## Dependency followups

Examined on 2026-10-18. No opportunities identified; no items or handoff prompts.

- Context: PR #920 merged on 2026-09-07 as `38a1dbd3be0bb0e6b1d28d52ec475a6482908cbb`.
- Module examined: direct Go dependency `google.golang.org/api v0.294.0 → v0.296.0`, covering intermediate release `v0.295.0` and the complete upstream commit range.
- Transitive changes: none. The PR changes no other module versions, and both versions of this module have the same `go.mod` checksum, so its dependency requirements are unchanged.
- Reason: the releases regenerate discovery clients that Prow does not import. Prow uses `option`, `iterator`, and `googleapi`; those packages and the shared transport/auth implementation are unchanged, apart from internal version metadata. No new deprecation, replacement API, useful feature, or default change applies to these call sites.
- Evidence: reused the saved dependency review and its release-note analysis; reconfirmed the manifest/checksum diff, upstream changed-file list, and Prow imports. Existing credential-loader deprecation comments in `pkg/io/credentials.go` concern the unchanged `cloud.google.com/go/auth` dependency and are not unlocked by this bump.
- Sources: [upstream comparison](https://github.com/googleapis/google-api-go-client/compare/v0.294.0...v0.296.0), [v0.295.0 release](https://github.com/googleapis/google-api-go-client/releases/tag/v0.295.0), [v0.296.0 release](https://github.com/googleapis/google-api-go-client/releases/tag/v0.296.0).
