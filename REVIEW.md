---
pr: kubernetes-sigs/prow#897
title: "chore(deps): bump github.com/fsouza/fake-gcs-server from 1.19.4 to 1.56.1"
head_sha: 43f78fe4e4d27628c0f29791bf12dd63f131a6df
base: main
reviewed_at: 2026-08-27T11:05:56Z
verdict: approve
---

## What this PR does

- Bumps direct test-only dependency `github.com/fsouza/fake-gcs-server` v1.19.4 → v1.56.1 (supersedes the abandoned dependabot PR #888, which only touched `go.mod`/`go.sum` and left CI broken).
- Adapts 23 `fakestorage.Object` struct literals across `cmd/deck/job_history_test.go`, `cmd/deck/pr_history_test.go`, `pkg/spyglass/spyglass_test.go`, and `test/integration/cmd/fakegcsserver/main.go` to the library's new layout, where `BucketName`/`Name`/`Metadata` moved into an embedded `ObjectAttrs`, `Content` stays on the outer `Object`.
- Pulls in transitive bumps via `go mod tidy` (`google.golang.org/api`, `grpc`, `testify`, `gorilla/handlers`, new `pkg/xattr`/`renameio/v2` indirects).
- Carries the dependency bump and the source fixes as a single commit, since splitting them would leave an intermediate commit that doesn't build.

## Dependency analysis

- **Freshness**: v1.56.1 is a real tagged release (not a pseudo-version), published 2026-08-24T03:23:45Z — only ~3 days before merge, which is within the "too fresh" band by the usual soak-time guideline. Mitigating factors: main was already forced onto this dependency's newer field layout by the stalled #888; v1.56.1 differs from the 2-days-older v1.56.0 only by trivial, low-risk commits (bucket-list pagination, a delete-by-move perf tweak, an ACL/generation fix, a Go 1.27 CI bump) — no meaningful extra soak time is bought by waiting.
- **Usage**: direct dependency, but exclusively test-only — grep confirms imports only in `_test.go` files and one test-fixture binary (`test/integration/cmd/fakegcsserver/main.go`). Not on any production/runtime code path.
- **Changelog & exposure**: ~1840 commits / 36 releases between v1.19.4 and v1.56.1. No CVEs or security fixes found in fake-gcs-server's own server code across the range; the "security" hits in the compare are all from fake-gcs-server's own transitive/example-dir dependency bumps (urllib3, jws, GitHub Actions), irrelevant to prow's import graph. Functional changes (precondition/generation handling, ACL/compose fixes, pagination) land in server behaviors prow's tests don't exercise. The one breaking change relevant to us — `Object` fields moving into embedded `ObjectAttrs` (~v1.20) — is exactly what the 23 literal updates in this PR address.
- **Take**: safe to take now despite the fresh tag, given the test-only exposure and that the jump was already forced by #888.

## Code review (struct-literal adaptation)

Reviewed all 23 changed `fakestorage.Object` literals across the four files diffed against `b4275468b` (merge-base). Every change is a mechanical 1:1 field move into `ObjectAttrs{BucketName, Name, Metadata}`, with `Content` correctly left on the outer struct — no behavioral change, matches the PR description exactly.

## Findings

None.

## Checked

- `go.mod`/`go.sum` diff: version bumps and transitive changes are consistent with a `go get` + `make update-go-deps` on this one direct dependency; no unrelated or unexplained module changes.
- All 23 `fakestorage.Object` literal call sites (`cmd/deck/job_history_test.go`, `cmd/deck/pr_history_test.go`, `pkg/spyglass/spyglass_test.go`, `test/integration/cmd/fakegcsserver/main.go`) — correct field placement, no dropped fields.
- `pkg/pod-utils/gcs/upload_test.go` — confirmed unchanged, as the PR description claims (doesn't use the affected literal fields).
- Import surface for `fsouza/fake-gcs-server` — test-only, no production usage.

## Open questions

None — PR already merged (2026-08-26) as `fbca53b5a`.

## Dependency followups

### [deprecation] Migrate `fakestorage.NewServer` to `NewServerWithOptions`

- **Unlocked by:** `github.com/fsouza/fake-gcs-server` v1.19.4 → v1.56.1 (PR #897, merged as `fbca53b5a`).
- **Where:** `pkg/spyglass/spyglass_test.go:86`, `pkg/spyglass/spyglass_test.go:1620`, `cmd/deck/job_history_test.go:363`, `cmd/deck/pr_history_test.go:582`, `pkg/pod-utils/gcs/upload_test.go:345` — five test-only call sites, all in files this bump already rewrote.
- **Necessity:** should — `fakestorage/server.go:68` in v1.56.1 carries `// Deprecated: use NewServerWithOptions.`; the marker already existed at v1.19.4, so the bump does not introduce it, but it is the only in-range deprecation that hits code we call (test-only, no behavior change, eventual-removal risk).
- **Mode:** deliver
- **Handoff prompt:**

```text
In `kubernetes-sigs/prow`, now that PR #897 ("chore(deps): bump
github.com/fsouza/fake-gcs-server from 1.19.4 to 1.56.1", merged as `fbca53b5a`)
has moved the module to v1.56.1, migrate the five remaining uses of the deprecated
`fakestorage.NewServer` to `fakestorage.NewServerWithOptions`.

Context: v1.56.1's `fakestorage/server.go` marks
`NewServer(objects []Object) *Server` with `// Deprecated: use NewServerWithOptions.`;
the wrapper just calls `NewServerWithOptions(Options{InitialObjects: objects})` and
discards the returned error.

Call sites (all test-only):
- `pkg/spyglass/spyglass_test.go:86` (inside `TestMain`)
- `pkg/spyglass/spyglass_test.go:1620`
- `cmd/deck/job_history_test.go:363`
- `cmd/deck/pr_history_test.go:582`
- `pkg/pod-utils/gcs/upload_test.go:345`

Task: replace each `fakestorage.NewServer(x)` with
`fakestorage.NewServerWithOptions(fakestorage.Options{InitialObjects: x})` and handle
the returned `error`:
- In `TestMain` (spyglass), log and exit non-zero (`log.Fatalf` or
  `fmt.Fprintln(os.Stderr, err); os.Exit(1)`), matching the file's existing style.
- In the four `Test*` functions, fail the test with `t.Fatalf` on error.

Acceptance criteria:
- `go build ./...` and `go vet ./pkg/spyglass/... ./cmd/deck/... ./pkg/pod-utils/gcs/...` pass.
- `go test ./pkg/spyglass/... ./cmd/deck/... ./pkg/pod-utils/gcs/...` passes.
- `rg 'fakestorage\.NewServer\(' --glob '!vendor/**'` returns no matches
  (only `NewServerWithOptions` remains).

Scope guard: no behavior changes; do not touch the `fakestorage.Object`/`ObjectAttrs`
literals, `defer …Stop()` lifecycle, or non-test code; no `NoListener` or other
`Options` changes; no other deprecation cleanup (`CreateBucket`, `ListObjects` are out
of scope); no dependency bumps.
```
