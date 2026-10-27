---
pr: kubernetes-sigs/prow#868
title: "chore(deps): bump github.com/tektoncd/pipeline from 1.6.2 to 1.15.0"
head_sha: 93d82ce35d4aad0868994a97864a59735f7b4de9
base: main
reviewed_at: 2026-08-25T12:01:55Z
verdict: approve
refresh_log:
  - from: 9400d97bf1da3ed7882d5a65769fb35827142b9a
    to: 93d82ce35d4aad0868994a97864a59735f7b4de9
    at: 2026-08-25T12:01:55Z
    summary: >
      PR merged. Head SHA changed because PR was rebased onto main (picked up
      several unrelated merged PRs: #859 Pub/Sub v2 migration, #865 aws group
      bump, #867 goldmark bump, #869 redigo bump). tektoncd/pipeline bump
      content unchanged (still v1.6.2 -> v1.15.0). Both previously-blocking
      findings resolved: go.mod/go.sum are tidy and pkg/spyglass error-string
      tests were fixed to use substring matching, tolerant of the
      cloud.google.com/go/storage bump's new error text -- fixed by #859
      (Migrate to Pub/Sub v2, commit 578f7668f), not by 868's author. CI is
      fully green as merged; approved by cblecker.
---

## Summary

Dependabot-style dep bump. Only `go.mod`/`go.sum` change (no vendor dir, no project code). Direct bump target is `github.com/tektoncd/pipeline v1.6.2 -> v1.15.0`; the rest of the diff is transitive `go mod tidy` fallout (cloud.google.com/go/*, go.opentelemetry.io/*, knative.dev/pkg, google.golang.org/api, sirupsen/logrus, go.uber.org/zap, hashicorp/go-retryablehttp, etc).

## Dependency analysis

- tektoncd/pipeline v1.15.0 released 2026-07-31 (real tag, not pseudo-version), 24 days old at review time. Fine on freshness; tagged LTS ("Toyger Orisa").
- Direct dep, heavily imported: 29 files under `cmd/pipeline/`, `pkg/apis/prowjobs/v1/`, `pkg/config/`, `pkg/pipeline/{clientset,informers,listers}/`. Prow only consumes v1 API types + generated client/informer/lister code (client of Tekton CRDs), does not run the Tekton controller/webhook/events-controller.
- Changelog v1.7.0->v1.15.0 is mostly controller-side (OTel metrics migration v1.10.0, CloudEvents moved to dedicated controller v1.12.0, resolver restrictions v1.13.0, configurable git-resolver backoff v1.15.0) — none apply to Prow's usage pattern.
- Several tektoncd/pipeline CVEs exist (CVE-2026-40161, -40923, -40938, -40924, -33022, -33211, -25542) but all were already fixed by v1.6.1/v1.6.2 (the PR's old version) — this bump is not a security-motivated fix for Prow.
- `cloud.google.com/go/storage` v1.50.0 -> v1.56.0 (direct, used in `pkg/spyglass`) changed credential-parsing error text from `"unknown credential type: %q"` to `credentials: unsupported filetype "..."` — breaks a test (see Findings).

## Findings

None currently open — see Resolved below.

## Resolved

### [blocking] go.mod/go.sum not tidy — verify-lint CI failing
- where: `go.mod`, `go.sum`
- concern: `pull-prow-verify-lint` failed at `head_sha: 9400d97b` — running `go mod tidy` in CI produced a diff against the committed files.
- resolution: by the time PR merged (`93d82ce3`), `pull-prow-verify-lint` passed cleanly. The PR was rebased onto a later `main` (picking up #859/#865/#867/#869); no separate tidy commit was needed for this to go green — the base rebase carried it.

### [blocking] TestSignURL/bad_type_errors broken by cloud.google.com/go/storage bump
- where: `pkg/spyglass/storageartifact_fetcher_test.go:422` (assertion), `pkg/spyglass/storageartifact_fetcher_test.go:475` (test body, `TestSignURL`)
- concern: the storage client library bump (v1.50.0 -> v1.56.0, transitive from the tektoncd/pipeline bump's `go mod tidy` resolution) changed the credential-parsing error message from `unknown credential type: %q` to `credentials: unsupported filetype "..."`. The test asserted the old exact string and failed on CI.
- resolution: fixed in commit `578f7668f` ("Migrate to Pub/Sub v2", PR #859) — merged to main before #868 was rebased, not authored by #868. The test now does substring matching (`strings.Contains`) instead of exact-string comparison, and the expected substrings were updated to the new wording (`credentials: unsupported filetype "user"`, etc). Confirmed via diff of `pkg/spyglass/storageartifact_fetcher_test.go` between the two reviewed SHAs.
- excerpt: |
    - if tc.err != err.Error() {
    + if tc.err == "" || !strings.Contains(err.Error(), tc.err) {
    -   err: "dialing: unknown credential type: \"user\"",
    +   err: `credentials: unsupported filetype "user"`,

### [should-fix] TestSize_GCS likely broken by same storage bump
- where: `pkg/spyglass/storageartifact_test.go:481-486` (`TestSize_GCS`/"Size of nonexistentArtifact")
- concern: failed alongside `TestSignURL` in the same CI run, same root cause suspected (error-wrapping format changed by the storage library bump).
- resolution: `pull-prow-unit-test` passes cleanly on the merged head (`93d82ce3`); `pkg/spyglass/storageartifact_test.go` also changed (34 lines) in the same `578f7668f` commit that fixed the sibling test above.

## Checked
- `pull-prow-image-build-test` and `pull-prow-integration` pass — code still compiles/integrates against the new tektoncd/pipeline v1 API types, no breaking API changes surfaced in `pkg/apis/prowjobs/v1/types.go` or `pkg/config/config.go` usage.
- `pkg/plugins/trick-or-treat TestReadImage` failure in the same CI run is unrelated — a real network fetch test (not gated behind `--human`), pre-existing flake, not caused by this bump.
- Only go.mod/go.sum changed; no vendor directory in this repo, so no vendored-code review needed.
- Other transitive bumps (logrus, zap, go-retryablehttp, cloud.google.com/go/* family, go.opentelemetry.io/*, knative.dev/pkg) are minor/patch version jumps, no flagged CVEs or behavior changes relevant to Prow.

## Open questions
- None outstanding — PR merged, CI green, findings resolved by an unrelated upstream fix (#859) that landed before this PR's rebase/merge.
