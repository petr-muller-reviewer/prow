---
pr: kubernetes-sigs/prow#941
title: "deck: merge job history from additional storage buckets"
head_sha: 76e01c4b1bd1b17db862076a4629ae401cd604d0
base: main
reviewed_at: 2026-09-23T20:39:17Z
verdict: request-changes
---

# Review

## Verdict

Request changes.

The merged-history implementation preserves primary-bucket precedence and degrades gracefully when an additional bucket cannot be listed. However, the advertised credential-backed configuration is not deployable with the provided Deck RBAC, and invalid credential configurations are accepted or can panic in local mode. Those paths need validation, RBAC/documentation, and tests before merge.

## What this PR does

- Adds `deck.spyglass.additional_history_buckets` for extra GCS or S3 locations to scan on each job-history request.
- Merges and deduplicates build IDs, retaining the bucket selected by the URL if an ID appears in multiple locations.
- Uses the source bucket for build metadata and Spyglass links.
- Supports per-additional-bucket credentials from Kubernetes Secrets and permits configured buckets through storage-path validation.

## Findings

### [blocking] Credential-backed buckets need Deck Secret RBAC
- where: `cmd/deck/main.go:696-710`
- concern: Selecting either credentials field reads a Secret during Deck startup and treats an authorization error as fatal. None of the starter Deck Roles grants access to core `secrets`, so the documented credential-backed configuration cannot start in a standard installation. Add the necessary narrowly scoped RBAC (and document the rollout/permission change) in the supported deployment manifests.
- excerpt: |
    case entry.GCSCredentialsSecret != nil:
        secret := &coreapi.Secret{}
        if err := k8sClient.Get(ctx, ctrlruntimeclient.ObjectKey{Namespace: cfg().ProwJobNamespace, Name: entry.GCSCredentialsSecret.Name}, secret); err != nil {
            return nil, fmt.Errorf("reading GCS credentials secret %q for bucket %q: %w", entry.GCSCredentialsSecret.Name, entry.Bucket, err)
        }

### [blocking] Validate the credential-selector contract
- where: `cmd/deck/main.go:695-710`
- concern: The configuration documents the two selectors as mutually exclusive, but the `switch` silently chooses GCS when both are set. It also accepts provider/selector mismatches and missing Secret keys, which become late failures or an opener with empty credentials. Validate URI/provider, zero-or-one selector, matching credential type, and selected key data before accepting the configuration; add tests for these cases.
- excerpt: |
    switch {
    case entry.GCSCredentialsSecret != nil:
        secret := &coreapi.Secret{}
        if err := k8sClient.Get(ctx, ctrlruntimeclient.ObjectKey{Namespace: cfg().ProwJobNamespace, Name: entry.GCSCredentialsSecret.Name}, secret); err != nil {
            return nil, fmt.Errorf("reading GCS credentials secret %q for bucket %q: %w", entry.GCSCredentialsSecret.Name, entry.Bucket, err)
        }
        bucketOpener, err = io.NewOpenerFromCredentialBytes(ctx, secret.Data[entry.GCSCredentialsSecret.Key], nil)
    case entry.S3CredentialsSecret != nil:
        secret := &coreapi.Secret{}

### [blocking] Do not dereference a nil client in local mode
- where: `cmd/deck/main.go:463-464`
- concern: In pregenerated-data mode, `k8sClient` is never initialized, yet Spyglass initialization still builds all configured additional buckets. Any credentials selector then calls `k8sClient.Get` and panics. Reject secret-backed buckets with a clear error in local mode, or inject a usable client.
- excerpt: |
    if o.spyglass {
        initSpyglass(cfg, o, mux, ja, githubClient, gitClient, k8sClient)
    }

### [should-fix] Bound the effect of unhealthy extra buckets
- where: `cmd/deck/job_history.go:462-479`
- concern: Each job-history request lists every configured extra bucket sequentially under the same ten-second deadline. A persistently slow bucket can consume the request budget for later buckets and emit warning logs on every request. Consider per-bucket limits or concurrency plus metrics/rate-limited logging.
- excerpt: |
    buildIDListCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()
    for _, ab := range additionalBuckets {
        ids, err := ab.listBuildIDs(buildIDListCtx, root)
        if err != nil {
            logrus.WithError(err).WithField("bucket", ab.getName()).Warning("Failed to list build IDs from additional bucket")
            continue
        }
    }

## Checked

- Empty `additional_history_buckets` returns without changing existing Deck initialization.
- Duplicate build IDs retain the URL-selected primary bucket, and later build-data and Spyglass-link reads use the recorded source bucket.
- Additional-list failures do not fail the entire history page.
- `git diff --check` passes.

## Open questions

- Which deployment manifests are the supported source for Deck RBAC, and can the Secret permissions be constrained to the configured credential Secret names?
- Is it acceptable that Secret rotation and configuration reloads require a Deck restart, or should credential-backed bucket clients participate in reload handling?
