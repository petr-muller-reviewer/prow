---
pr: kubernetes-sigs/prow#954
title: "docs: repair the dead links in the site content"
head_sha: d7e3bd598e2848e54b1e6e95fca30b778edc8346
base: main
reviewed_at: 2026-09-23T10:48:46Z
verdict: approve
refresh_log:
  - old_sha: 83f680a7c1e75d9188225654e55c0101fb0c974a
    new_sha: 83f680a7c1e75d9188225654e55c0101fb0c974a
    at: 2026-09-22T13:47:43Z
    summary: "No code changes; Prucek independently reported the existing dead RBAC-manifest link."
  - old_sha: 83f680a7c1e75d9188225654e55c0101fb0c974a
    new_sha: d7e3bd598e2848e54b1e6e95fca30b778edc8346
    at: 2026-09-23T10:48:46Z
    summary: "Resolved the controller-manager RBAC finding and repaired two additional dead links."
---

## Verdict

Approve. The controller-manager RBAC concern is resolved, and the two additional links use reachable canonical sources. The remaining choices identified by the author are maintainer policy questions rather than defects in these changes.

## What this PR does

- Replaces obsolete live links with current canonical sources.
- Pins historical `test-infra` references to immutable pre-removal commits.
- Updates generic-autobumper examples to the current jobs file without line anchors.
- Changes discontinued adopter subpaths to their project roots.

Since previous review:

- No commits or diff changes were added; the PR remains open at `83f680a7c1e75d9188225654e55c0101fb0c974a`.
- Prucek left an inline comment on `site/content/en/docs/components/core/prow-controller-manager.md:34` reporting the same dead RBAC-manifest link, and submitted a COMMENTED review.

Since previous review:

- `ce331cee9` replaced the separate controller-manager deployment and dead RBAC links with one reachable combined manifest.
- `d7e3bd598` repaired the deprecated `tackle` CRD URL and the metrics pushgateway manifest URL; the three new targets return HTTP 200.
- Prucek approved the PR; the author also documented two unresolved maintainer-policy questions about example URLs.

## Findings

No unresolved findings.

## Resolved

### [should-fix] Controller-manager RBAC link was still dead
- where: `site/content/en/docs/components/core/prow-controller-manager.md:33-34`
- concern: The page formerly linked a separate RBAC manifest that no longer exists in `kubernetes/test-infra`.
- resolution: Commit `ce331cee9` replaces the two bullets with the reachable `kubernetes/k8s.io` manifest, which contains the deployment and RBAC resources.

## Checked

- Historical announcement and secrets links use immutable `test-infra` commit SHAs appropriate to their time-specific context.
- Current ghproxy and controller-manager deployment links point to `kubernetes/k8s.io` manifests.
- Generic-autobumper examples point to the current jobs file and no longer rely on fragile line anchors.
- The new controller-manager, tackle CRD, and pushgateway targets each return HTTP 200.
- This is Markdown-only: no Prow configuration, runtime behavior, or upgrade compatibility changes.

## Open questions

- Should `generic-autobumper.md` use a different repository for its `upstreamURLBase` example now that `test-infra` no longer carries Prow configuration?
