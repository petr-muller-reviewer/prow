---
pr: kubernetes-sigs/prow#954
title: "docs: repair the dead links in the site content"
head_sha: 83f680a7c1e75d9188225654e55c0101fb0c974a
base: main
reviewed_at: 2026-09-21T16:10:59Z
verdict: request-changes
refresh_log:
  - old_sha: 83f680a7c1e75d9188225654e55c0101fb0c974a
    new_sha: 83f680a7c1e75d9188225654e55c0101fb0c974a
    at: 2026-09-22T13:47:43Z
    summary: "No code changes; Prucek independently reported the existing dead RBAC-manifest link."
---

## Verdict

Request changes. The changed targets are sound, but the controller-manager page still contains a dead RBAC-manifest link, leaving an operator-facing dead end in a PR intended to repair site links.

## What this PR does

- Replaces obsolete live links with current canonical sources.
- Pins historical `test-infra` references to immutable pre-removal commits.
- Updates generic-autobumper examples to the current jobs file without line anchors.
- Changes discontinued adopter subpaths to their project roots.

Since previous review:

- No commits or diff changes were added; the PR remains open at `83f680a7c1e75d9188225654e55c0101fb0c974a`.
- Prucek left an inline comment on `site/content/en/docs/components/core/prow-controller-manager.md:34` reporting the same dead RBAC-manifest link, and submitted a COMMENTED review.

## Findings

### [should-fix] Controller-manager RBAC link is still dead
- where: `site/content/en/docs/components/core/prow-controller-manager.md:34`
- concern: The adjacent deployment link was updated, but this RBAC link still targets `kubernetes/test-infra` `master`, where the referenced file no longer exists. Readers following the deployment guidance cannot retrieve the separately linked RBAC manifest. Replace it with a valid canonical resource or remove it if the RBAC resources embedded in the new deployment manifest are the intended replacement.
- excerpt: |
    * [Deployment manifest](https://github.com/kubernetes/k8s.io/blob/main/kubernetes/gke-prow/prow/prow-controller-manager.yaml)
    * [RBAC manifest](https://github.com/kubernetes/test-infra/blob/master/config/prow/cluster/prow_controller_manager_rbac.yaml)

## Checked

- Historical announcement and secrets links use immutable `test-infra` commit SHAs appropriate to their time-specific context.
- Current ghproxy and controller-manager deployment links point to `kubernetes/k8s.io` manifests.
- Generic-autobumper examples point to the current jobs file and no longer rely on fragile line anchors.
- This is Markdown-only: no Prow configuration, runtime behavior, or upgrade compatibility changes.

## Open questions

- Which canonical RBAC resource should replace the standalone controller-manager RBAC link, or should the page direct readers to the RBAC embedded in `prow-controller-manager.yaml`?
