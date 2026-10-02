---
pr: kubernetes-sigs/prow#984
title: "deck: fix a panic and retries on GCS errors"
head_sha: 440fb4f15287c02423612c0ee21f3f5bac6ea11d
base: main
reviewed_at: 2026-10-02T10:22:35Z
verdict: request-changes
---

# Review

## Verdict

Request changes for two gaps in the fixes. The lens server still accepts an in-range index that now names a different lens after config reload, and the path simplifier leaves documented presubmit URL forms unmatched.

## What this PR does

- Validates the lens index in Deck and the lens server.
- Extends Deck's request path labels for common job and history routes.
- Stops retrying permanent GCS listing failures while retaining retries for transient errors.

## Findings

### [should-fix] Verify the lens identity after config reload
- where: `pkg/spyglass/lenses/common/common.go:118-123`
- concern: The new guard rejects indexes that are out of range, but an in-range index can point to a different lens after the config is reordered or replaced. The handler then renders this lens with the other lens's config, so the backend check does not fully protect the config reload race described in the PR; compare the selected config's name with `opts.LensName` as Deck does.
- excerpt: |
    lenses := opts.ConfigGetter().Deck.Spyglass.Lenses
    if request.LensIndex < 0 || request.LensIndex >= len(lenses) {
        writeHTTPError(w, fmt.Errorf("invalid lens index %d, %d lenses are configured", request.LensIndex, len(lenses)), http.StatusBadRequest)
        return
    }
    lensConfig := lenses[request.LensIndex].Lens.Config

### [should-fix] Cover existing presubmit URL forms in the path labels
- where: `cmd/deck/main.go:277-279`
- concern: This only matches pull paths with an `<org_repo>` segment and no trailing slash. The repository documents `/view/gcs/kubernetes-jenkins/pr-logs/pull/93714/pull-kubernetes-node-e2e/1291409525907132416/`, and existing Spyglass cases support the form without an org or repo segment. Those requests still resolve to `unmatched`, so the path metrics fix is incomplete for supported presubmit links.
- excerpt: |
    l("pr-logs",
        l("pull", v("org_repo", v("pr", v("job", v("build"))))),
        l("directory", v("job", v("build")))),

## Checked

- Deck rejects negative, out-of-range, and mismatched lens indexes before proxying.
- The GCS classifier covers missing buckets and objects, plus API 401, 403, and 404 responses; other errors retain the retry path.
- The simplifier tests cover the newly added route shapes without trailing slashes.
- Standards review found no applicable local style violations or clear baseline smells.

## Open questions

- None.
