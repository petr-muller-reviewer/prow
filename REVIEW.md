---
pr: kubernetes-sigs/prow#1071
title: "chore(deps): bump cloud.google.com/go/secretmanager from 1.23.0 to 1.24.0"
head_sha: 6ceb2e6e750898d7399907e2200d59fdb30dc6fc
base: main
reviewed_at: 2026-11-16T23:14:02Z
verdict: approve
---

# Review

## Verdict

Safe to merge now: dep-only bump, no advisories either side, and every changed API surface is in code Prow never calls.

`go.mod`/`go.sum` are the only changed files and `go mod tidy -diff` is clean. secretmanager 1.24.0 was published 2026-11-05 (11 days old) — past the too-fresh threshold, short of the two-week mark. OSV reports no advisories for the module at 1.23.0 or 1.24.0, and `govulncheck` finds the same 42 pre-existing findings at base and head. The 1.24.0 regeneration renames the managed-rotation request field and nested message type (source-incompatible) and adds 60s default timeouts there; Prow calls only CreateSecret/GetSecret/ListSecrets/UpdateSecret/AddSecretVersion/AccessSecretVersion, so nothing we compile against moves. `cmd/webhook-server` builds and all PR checks pass.

## What this PR does

- Bumps the direct dependency `cloud.google.com/go/secretmanager` from v1.23.0 to v1.24.0 (dependabot).
- Pulls the matching MVS fallout from secretmanager's own `go.mod`: `cloud.google.com/go/iam` 1.13.0 → 1.14.0, `longrunning` 1.2.0 → 1.3.0, `googleapis/gax-go/v2` 2.26.2 → 2.27.0.
- Moves `go.opentelemetry.io/otel` (and metric/trace/sdk/sdk-metric) 1.46.0 → 1.47.0 and adds `go.opentelemetry.io/otel/log` v1.47.0 as a new indirect.
- Moves the direct `google.golang.org/genproto` and `genproto/googleapis/api` pseudo-versions from `20260904194346-d0f1323225a4` to `20261012220534-4341edf3f941`.
- No source, config, documentation, or test changes.

## Findings

None.

## Checked

- Classification: dep-only — `git diff --stat` shows only `go.mod` and `go.sum`.
- Freshness: `secretmanager v1.24.0` published 2026-11-05T19:01Z (11 days). Indirects: gax v2.27.0 2026-10-21, iam v1.14.0 and longrunning v1.3.0 2026-09-24, otel v1.47.0 2026-10-02.
- OSV: no advisories for `cloud.google.com/go/secretmanager` at 1.23.0 or 1.24.0.
- `govulncheck` module scans at merge-base and head: same two module-level findings, `GO-2023-1901` and `GO-2026-5932`.
- `govulncheck` symbol scans (`./cmd/... ./pkg/...`) at base and head: 42 identical findings, none in secretmanager or in any module bumped here. `GO-2023-1901` (tektoncd/pipeline, called via `pkg/apis/prowjobs/v1` deep-copies) and `GO-2026-5932` (x/crypto/openpgp, module-level only, not imported) both pre-date this PR.
- Usage surface: two imports, both in `cmd/webhook-server/secretmanager/secretmanager.go:23-24`; wired up at `cmd/webhook-server/main.go:151` and `cmd/webhook-server/clients.go:32`. No other package imports it.
- Changelog 1.23.1 → 1.24.0: 1.23.1 is "additional validation for REST RPC requests" (gax.ValidatePathExpressions on `parent`/`name` in the generated REST transport); 1.24.0 is "update API sources and regenerate".
- Byte-level diff of the module zips: only `secret_manager_client.go`, `resources.pb.go`, `service.pb.go` (plus version/CHANGES/go.mod) change; the RPCs Prow calls keep their signatures.
- Managed-rotation API is unused: no references to `EnableManagedRotation`, `RotateSecret`, or `CloudSqlSingleUserCredentials` outside vendor, so the `Credentials` → `Credential` rename cannot break this repo.
- `go build ./cmd/webhook-server/...` passes at head.
- `go mod tidy -diff` produces no output, so the manifests are tidy.
- `go.sum` additions are all `cloud.google.com/go`, `go.opentelemetry.io`, `google.golang.org`, `github.com/googleapis` — no unexpected hosts. The `cloud.google.com/go/logging` hashes enter only through genproto's module graph and are not built.
- CI at head: `pull-prow-unit-test`, `pull-prow-integration`, `pull-prow-verify-lint`, `pull-prow-image-build-test`, race-detector job all pass; tide waits only on lgtm/approved.

## Open questions

- secretmanager 1.24.0 is 11 days old; is that enough soak for this repo's taste, or hold a few more days?
- 1.24.0 renames `EnableManagedRotationRequest.Credentials`/`..._CloudSQLSingleUserCredentials` and adds default 60s timeouts on managed-rotation RPCs — a source-incompatible change in a minor release. Harmless for us, but worth knowing if Prow ever exposes managed rotation.
- `google.golang.org/genproto` stays a direct requirement only because `cmd/webhook-server/secretmanager/secretmanager.go:26` imports the legacy `google.golang.org/genproto/protobuf/field_mask`; switching to `google.golang.org/protobuf/types/known/fieldmaskpb` would drop that to indirect. Out of scope here.

## Dependency followups

No opportunities identified (run on PR #1071 head `6ceb2e6`, PR still open).

- `cloud.google.com/go/secretmanager` v1.23.0 → v1.24.0 (direct; imported at `cmd/webhook-server/secretmanager/secretmanager.go:23-24`): the only API-surface change is managed rotation — `EnableManagedRotationRequest.Credentials` → `Credential`, the nested `…_CloudSQLSingleUserCredentials` type promoted to top-level `CloudSqlSingleUserCredentials`, and new 60s default timeouts on `EnableManagedRotation`/`RotateSecret`. Prow calls none of those RPCs, and no deprecation we call was added (the three `Deprecated:` markers in the generated client — `Connection()` on the client/gRPC/REST transports — already existed in 1.23.0 and we never call them). v1.23.1's REST path validation does not apply to our gRPC client.
- `google.golang.org/genproto` / `genproto/googleapis/api` `20260904194346-d0f1323225a4` → `20261012220534-4341edf3f941` (direct, via the legacy `protobuf/field_mask` import at `secretmanager.go:26`): the eight intervening commits are "update all" dependency housekeeping, and `protobuf/field_mask` is byte-identical between the two revisions (still an alias to `fieldmaskpb`, not newly deprecated). Migrating that import is a pre-existing tidy-up, not something this bump unlocks.
- `googleapis/gax-go/v2` 2.26.2 → 2.27.0 (indirect) — adds the path-expression helper used by regenerated clients; Prow does not import gax.
- `cloud.google.com/go/iam` 1.13.0 → 1.14.0, `longrunning` 1.2.0 → 1.3.0 (indirect) — not imported by Prow.
- `go.opentelemetry.io/otel` 1.46.0 → 1.47.0 plus new `otel/log` v1.47.0 (indirect) — the release's additions (log API stability, attribute depth limits, metric `WithMaxExportBatchSize`, sampler/metric extension interfaces) would need direct otel usage; Prow imports none of it.
