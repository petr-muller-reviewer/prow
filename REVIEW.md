---
pr: kubernetes-sigs/prow#859
title: "Migrate to Pub/Sub v2"
head_sha: 578f7668f880883394c56d9da777c27b119dd79a
base: main
reviewed_at: 2026-08-25T15:25:07Z
verdict: approve
pr_state: merged
refresh_log:
  - at: 2026-08-25T15:25:07Z
    old_sha: 578f7668f880883394c56d9da777c27b119dd79a
    new_sha: 578f7668f880883394c56d9da777c27b119dd79a
    summary: No code changes. ameukam left /lgtm /hold (2026-08-24T10:20:39Z); cblecker cleared the hold (2026-08-24T19:42:03Z) and PR was merged.
---

## Summary

- Migrates `pkg/pubsub` (and callers) from `cloud.google.com/go/pubsub` v1 client API to `cloud.google.com/go/pubsub/v2`.
- Replaces `ReceiveSettings.Synchronous = true` with `NumGoroutines = 1` on the subscriber, relying on v2's default client-wide flow controller (keyed on `MaxOutstandingMessages`, shared across streams) for fairness instead of the removed `Synchronous` mode.
- Adds shared test fixture helpers (`pkg/testutil/fixtures.go`: `WriteCredentialsFile`, `WriteAuthorizedUserCredentialsFile`) and starts using them in some tests.
- Build and full test suite pass; one pre-existing test panic in `pkg/pubsub/subscriber/subscriber_test.go` verified to also occur on the pre-PR commit (out of scope for this PR).

## Findings

### [nit] Comment misattributes fairness guarantee to NumGoroutines
- where: `pkg/pubsub/subscriber/server.go:98`
- concern: The comment justifying `NumGoroutines = 1` ("Use one pull stream so multiple subscriber replicas can share work fairly") attributes fairness to the stream count. In pubsub/v2, fairness actually comes from the default (non-per-stream) client-wide flow controller keyed on `MaxOutstandingMessages`, which applies regardless of `NumGoroutines`. Pinning `NumGoroutines = 1` instead mainly guards against the library's documented future default flip of `EnablePerStreamFlowControl` to `true` (which would multiply the outstanding-message cap by `NumGoroutines`). A future contributor could read the current comment, conclude raising `NumGoroutines` for throughput is safe, and reintroduce the multiplied-cap problem once that default flips.
- excerpt: |
    // Use one pull stream so multiple subscriber replicas can share work fairly.
    NumGoroutines: 1,

### [nit] Duplicated credentials-fixture logic instead of reusing new test helper
- where: `pkg/io/opener_test.go:134`
- concern: This PR adds `pkg/testutil/fixtures.go` (`WriteCredentialsFile`/`WriteAuthorizedUserCredentialsFile`) and uses it earlier in `opener_test.go`, but `Test_opener_SignedURL` a few lines below still hand-rolls an equivalent `os.CreateTemp`+`Write`+`Close` credentials fixture. The same duplicate pattern remains in `pkg/spyglass/storageartifact_fetcher_test.go` (~line 443), a file this PR already edits nearby for unrelated error-string updates. Leaves three independent implementations of the same fake-credentials-file fixture that can drift out of sync.
- excerpt: |
    f, err := os.CreateTemp(t.TempDir(), "creds-*.json")
    ...
    f.Write(...)
    f.Close()

## Checked
- Build and full test suite pass on PR head.
- Pre-existing panic in `pkg/pubsub/subscriber/subscriber_test.go` reproduced on pre-PR commit too — unrelated to this change.
- Verified (by reading vendored `cloud.google.com/go/pubsub/v2@v2.6.0` source) that replacing `Synchronous=true` with `NumGoroutines=1` does not regress per-replica fairness: `MaxOutstandingMessages` is enforced by a client-wide flow controller shared across streams by default, independent of `NumGoroutines`.

## Open questions
- Would you consider rewording the `NumGoroutines = 1` comment to reference the future `EnablePerStreamFlowControl` default flip rather than implying the fairness guarantee comes from the stream count itself?
- Any interest in a quick follow-up to converge `opener_test.go`'s `Test_opener_SignedURL` and `storageartifact_fetcher_test.go` onto the new `testutil` credential-fixture helpers, now that they exist?

## Activity since previous review (2026-08-18T21:04:05Z)
- 2026-08-24T10:20:39Z: ameukam (MEMBER) commented `/lgtm` `/hold`.
- 2026-08-24T19:42:03Z: cblecker (MEMBER) commented `/hold cancel`, thanking ameukam.
- PR merged (head SHA unchanged at 578f7668f880883394c56d9da777c27b119dd79a). Neither nit finding was addressed before merge; they remain valid as follow-up candidates.

## Followups

Recorded 2026-11-01, post-merge of `d5c03da058aa2ceccffc1346d6948dab1d183b9c` (PR head `578f7668f880883394c56d9da777c27b119dd79a`). All four are `Mode: deliver`.

### [should] Make the subscriber fairness guard explicit instead of a misleading comment
- category: deferred-review
- where: `pkg/pubsub/subscriber/server.go:95-100`
- mode: deliver
- necessity: should — the unaddressed nit from the merge review; without it, a future throughput tweak can silently reintroduce the multiplied-cap problem when v2 flips `EnablePerStreamFlowControl` on.
- why followup: it is a comment/API-clarity fix, not a merge blocker, so it did not belong in the migration PR.

```
In kubernetes-sigs/prow, following the merge of PR #859 ("Migrate to Pub/Sub v2", merge
commit d5c03da058aa2ceccffc1346d6948dab1d183b9c), make the subscriber's fairness guard
explicit in `pubSubClient.subscription` in `pkg/pubsub/subscriber/server.go` (currently
lines 95-100).

Background: PR #859 replaced v1's `ReceiveSettings.Synchronous = true` with
`NumGoroutines = 1`. In cloud.google.com/go/pubsub/v2@v2.6.0, `NumGoroutines` already
defaults to 1 (`DefaultReceiveSettings`), and per-replica fairness comes from the default
client-wide flow controller keyed on `MaxOutstandingMessages`.
`EnablePerStreamFlowControl` defaults to false but is documented to default to true in the
next major version; with that on, `MaxOutstandingMessages` is enforced per stream, so the
effective cap becomes cap x `NumGoroutines` and one replica can take more than its share.

Task:
1. Replace the comment `// Use one pull stream so multiple subscriber replicas can share
   work fairly.` with one that states (a) fairness comes from the shared, client-wide
   `MaxOutstandingMessages` flow controller, and (b) `EnablePerStreamFlowControl` must stay
   off, because per-stream enforcement would multiply the cap by `NumGoroutines` and let a
   single replica exhaust the shared budget.
2. Set `sub.ReceiveSettings.EnablePerStreamFlowControl = false` explicitly alongside
   `MaxOutstandingMessages` and `NumGoroutines = 1`, so a future library default flip
   cannot change Prow's behaviour silently.
3. Add `pkg/pubsub/subscriber/server_test.go` with a test that builds a client against
   `cloud.google.com/go/pubsub/v2/pstest` (pattern: `newTestReporter` in
   `pkg/crier/reporters/pubsub/reporter_test.go`), calls
   `(&pubSubClient{client: client}).subscription("sub", 7)`, type-asserts the result to
   `*pubSubSubscription` and asserts `sub.ReceiveSettings.MaxOutstandingMessages == 7`,
   `NumGoroutines == 1`, and `EnablePerStreamFlowControl == false`.

Acceptance criteria: `go build ./pkg/pubsub/...` passes and
`go test ./pkg/pubsub/subscriber/ -run TestSubscription` passes (use `-run`; the package's
`TestHandleMessage` currently panics for unrelated reasons — separate followup). No
behaviour change at runtime.

Scope guard: do not change `NumGoroutines`, the `MaxOutstandingMessages` config plumbing,
the receive loop, or `pkg/config`. Do not fix the `TestHandleMessage` panic here.
```

### [should] Stop `TestHandleMessage` from segfaulting on ignored setup errors
- category: tests
- where: `pkg/pubsub/subscriber/subscriber_test.go:340-341` (duplicate setup at `:586-587`)
- mode: deliver
- necessity: should — a nil-pointer panic aborts the whole package's test run, so this package currently gives no signal in environments where the git client factory fails; the PR touched this file, so fixing it now is natural.
- why followup: the panic predates PR #859 (verified during review and reproduced again on 2026-11-01), so it was out of scope for the migration.

```
In kubernetes-sigs/prow, following PR #859 ("Migrate to Pub/Sub v2", merge commit
d5c03da058aa2ceccffc1346d6948dab1d183b9c), fix the nil-pointer panic in
`pkg/pubsub/subscriber/subscriber_test.go`.

Both `TestHandleMessage` (lines ~340-341) and the second call site (lines ~586-587) do:

    gitClient, _ := (&flagutil.GitHubOptions{}).GitClientFactory("abc", nil, true, false)
    cache, _ := config.NewInRepoConfigCache(100, ca, gitClient)

Both errors are discarded. When `GitClientFactory` fails (it does in a plain checkout with
no GitHub credentials), `cache` is a nil `*config.InRepoConfigCache`, and the
`PresubmitForGitHub` subtest panics in
`config.(*InRepoConfigCache).GetProwYAMLWithoutDefaults` (pkg/config/cache.go:358) via
`gangway.HandleProwJob` (pkg/gangway/gangway.go:917), aborting the entire package run
instead of failing one subtest.

Task:
1. Capture both errors and fail fast with `t.Fatalf`/`t1.Fatalf` and a readable message,
   e.g. `t.Fatalf("create git client factory: %v", err)` /
   `t.Fatalf("create in-repo config cache: %v", err)`.
2. Factor the shared setup into a small helper used by both call sites so the two copies
   cannot drift.

Acceptance criteria: `go test ./pkg/pubsub/subscriber/` never panics; it either passes or
reports the setup error against the affected test. No test is skipped or deleted.

Scope guard: do not change `Subscriber`, `handleMessage`, `pkg/gangway`, or
`pkg/config`'s caching behaviour. Do not build new fakes to make `GitClientFactory`
succeed — the deliverable is graceful failure, not new test infrastructure. Leave the
`RequireTenantID`/`allowedApiClient` arguments at the second call site alone.
```

### [could] Converge duplicated fake-credentials fixtures onto the new testutil helpers
- category: deferred-review
- where: `pkg/io/opener_test.go:131-150`, `pkg/spyglass/storageartifact_fetcher_test.go:440-456`
- mode: deliver
- necessity: could — cosmetic test hygiene; three independent copies of the same fixture can drift, but nothing is broken today.
- why followup: both files were edited by PR #859 but these blocks were left hand-rolled; a test-only cleanup would have padded the migration diff.

```
In kubernetes-sigs/prow, following PR #859 ("Migrate to Pub/Sub v2", merge commit
d5c03da058aa2ceccffc1346d6948dab1d183b9c), converge the hand-rolled fake-credentials-file
fixtures onto the helpers that PR added in `pkg/testutil/fixtures.go`
(`WriteCredentialsFile`, `WriteAuthorizedUserCredentialsFile`).

The duplicated block appears in `Test_opener_SignedURL`
(`pkg/io/opener_test.go:131-150`) and in the spyglass signed-URL test
(`pkg/spyglass/storageartifact_fetcher_test.go:440-456`): `os.CreateTemp("", "fake-creds")`,
`fp.Write([]byte(...))`, `fp.Close()`, `defer os.Remove(...)`.

Task:
1. Replace each block with `testutil.WriteCredentialsFile(t, tt.fakeCreds)` (or the local
   table field name) inside the existing `if fakeCreds != ""` guard, assigning the returned
   path; the helper already writes into `t.TempDir()` and cleans up, so drop the manual
   remove/close.
2. Remove imports/variables that become unused in the two files.

Acceptance criteria: `go test ./pkg/io/... ./pkg/spyglass/...` passes and `gofmt -l` is
clean on both files; `rg 'os.CreateTemp\("", "fake-creds"\)'` returns nothing.

Scope guard: only these two test files. Do not change `pkg/testutil`'s helper signatures or
production code. Leave unrelated `os.CreateTemp` uses (secret agent, interrupts,
statusreconciler, pod-utils) alone.
```

### [could] Cache the crier Pub/Sub client/publisher instead of building one per report
- category: tech-debt
- where: `pkg/crier/reporters/pubsub/reporter.go:114-133`
- mode: deliver
- necessity: could — saves repeated gRPC dial/auth and publisher goroutine churn on every reported ProwJob; no correctness problem today.
- why followup: the `TODO: Consider caching the pubsub client.` is pre-existing and the per-report lifecycle is unchanged by the migration, so it is a performance/style improvement rather than a fix.

```
In kubernetes-sigs/prow, following PR #859 ("Migrate to Pub/Sub v2", merge commit
d5c03da058aa2ceccffc1346d6948dab1d183b9c), remove the per-report client/publisher
construction in `Client.Report` (`pkg/crier/reporters/pubsub/reporter.go:114-133`, which
still carries `// TODO: Consider caching the pubsub client.`).

Today every report calls `newPubSubClient`, creates `client.Publisher(topic)`, publishes
one message, `publisher.Stop()`s, and `client.Close()`s. Since v2, `Publisher` owns
batching goroutines and `Close()` tears down the gRPC connection plus the
`TopicAdminClient` used on the publish-failure path, so the per-report cost is real.

Task:
1. Cache one `*pubsub.Client` per project (and, if it keeps the error paths intact, one
   `*pubsub.Publisher` per project/topic) on `Client`, created lazily under a mutex/singleflight.
2. Keep the injectable `newPubSubClient` seam so `reporter_test.go` can still point at a
   `pstest` server; make the cache test-overridable (e.g. reset it in `newTestReporter`).
3. If crier offers no shutdown hook to call `Client.Close`/`Publisher.Stop`, keep the
   cached objects for the process lifetime (as other long-lived clients do) and say so in a
   comment rather than inventing a lifecycle.

Acceptance criteria: `go test ./pkg/crier/reporters/pubsub/...` and
`go test ./pkg/crier/...` pass; the three existing `TestReport` cases (existing topic,
missing topic => user error, publish PermissionDenied => non-user error) still pass
unchanged; a second `Report` call for the same project does not dial a new client (cover
with a counter around `newPubSubClient`).

Scope guard: do not change `ReportMessage`, metric names, the NotFound =>
`criercommonlib.UserError` classification, or the 10s report timeout. Do not touch the
subscriber side.
```
