---
pr: kubernetes-sigs/prow#963
title: "chore(deps): bump google.golang.org/grpc from 1.83.1 to 1.83.2 in /hack/tools"
head_sha: 02dea49858aba338a3008af775b697b1a7dd34a9
base: main
reviewed_at: 2026-11-13T17:27:11Z
verdict: approve
refresh_log:
  - at: 2026-11-13T17:27:11Z
    old_sha: 02dea49858aba338a3008af775b697b1a7dd34a9
    new_sha: 02dea49858aba338a3008af775b697b1a7dd34a9
    summary: "No code changes; petr-muller approved at 12:33Z, Prow merged as 695a330 at 12:52Z, dependabot deleted the head branch."
---

## Verdict

approve — This is a scoped, 29-day-old indirect dependency update for the developer-tools module. grpc-go v1.83.2 hardens server handling of malformed HTTP/2 authority headers; Prow's `hack/tools` reaches grpc through controller-gen and does not directly import or run a gRPC server. The PR has since been merged (2026-09-23T12:52:13Z, merge commit `695a330`).

## What this PR does

- Updates `google.golang.org/grpc` in the standalone `hack/tools` Go module.
- Moves the selected version from `v1.83.1` to `v1.83.2`.
- Updates only the corresponding module checksums.
- Does not alter Prow production code, tests, configuration, or the root Go module.

Since previous review:

- No code changes — head is still `02dea49`; no new commits, comments, or inline review threads.
- `petr-muller` submitted an approving review at 2026-09-23T12:33:22Z (after `reviewed_at`), matching the verdict recorded here.
- Prow's approval notifier recorded the approval and merged the PR at 2026-09-23T12:52:13Z as `695a330` (merge into `main`); dependabot then deleted the head branch.
- `kubevirt-bot` cross-referenced the PR on 2026-09-25T06:10:39Z — an external reference event, no action needed.

## Findings

None.

## Checked

- `hack/tools/go.mod:351` marks the bump indirect; `go mod why` resolves it through `controller-gen`, Kubernetes API-extension validation, and apiserver egress-selector code.
- `hack/tools` has no direct Go import of `google.golang.org/grpc`; the PR changes only `hack/tools/go.mod` and `hack/tools/go.sum`.
- grpc-go v1.83.2 was released 2026-08-25 (29 days before review), is a tagged release, and its relevant change rejects server requests missing both `:authority` and `Host` headers.
- The remaining release change adjusts grpc-go's own OpenTelemetry tests for Go 1.27 gzip output; no applicable API or behavior change was found for the tools path.
- `cd hack/tools && go mod verify` passed: `all modules verified`.

## Open questions

None.

## Dependency followups

No improvement opportunities identified. Examined `google.golang.org/grpc` `v1.83.1` → `v1.83.2` (the PR's only version change; indirect in `hack/tools`). The release contains one security fix (servers reject requests missing both `:authority` and `Host` headers) and an internal `x/net` version refresh for Go 1.27 compatibility — no deprecations, API additions, or default changes. The `hack/tools` module contains no Go import of `google.golang.org/grpc` (0 matches), and the root module, which does import it (`pkg/gangway`, `pkg/resultstore`, `pkg/crier/reporters/pubsub`), is not part of this bump.
