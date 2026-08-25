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
