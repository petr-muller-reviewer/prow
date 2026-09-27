---
pr: kubernetes-sigs/prow#954
title: "docs: repair the dead links in the site content"
head_sha: d7e3bd598e2848e54b1e6e95fca30b778edc8346
base: main
reviewed_at: 2026-09-27T15:36:24Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-27T15:36:24Z
  gated_head_sha: d7e3bd598e2848e54b1e6e95fca30b778edc8346
  reviewed_head_sha: d7e3bd598e2848e54b1e6e95fca30b778edc8346
---

## Gate

Merge. The earlier inline RBAC concern is resolved. The two replacement targets discussed below are imperfect examples, but both replace dead links in existing documentation, and neither changes how Prow runs or creates a new deployment regression.

Gating list: None.

The autobumper postsubmit example at `site/content/en/docs/components/cli-tools/generic-autobumper.md:25` runs a Terraform deployment, and the linked pushgateway file at `site/content/en/docs/metrics/_index.md:72` is commented out. These are optional documentation clarifications, not reasons to hold this link-repair PR. The author explicitly offered the postsubmit choice for reviewer input, and the PR subsequently received an approval.

Independent merge risk: No notable API, configuration, or runtime compatibility risk. The PR changes only 11 site Markdown files; it does not affect existing Prow deployments. No repository skill in the available list applies to these link-only changes.

## Verdict

Approve.

The PR repairs the dead links in scope. The examples noted below could be clearer, but their limitations do not justify blocking a documentation-only link repair.

## What this PR does

- Pins historical `test-infra` documentation references to commits from before their removal.
- Redirects current Prow component links to their new locations in `kubernetes/k8s.io` or this repository.
- Updates the generic-autobumper job examples to a current jobs file.
- Replaces discontinued adopter subpaths with their project roots.

## Findings

No unresolved findings.

## Resolved

### [should-fix] Controller-manager RBAC link was still dead
- where: `site/content/en/docs/components/core/prow-controller-manager.md:33`
- concern: The earlier review found that the separate RBAC manifest link no longer existed in `kubernetes/test-infra`.
- resolution: Commit `ce331cee9` replaces the deployment and RBAC links with a reachable combined manifest in `kubernetes/k8s.io` that contains the deployment and RBAC resources.

## Checked

- All 18 new GitHub and raw GitHub targets in the site diff returned HTTP 200 on 2026-09-27.
- The current controller-manager target contains deployment, service account, role, and role binding resources.
- The new periodic autobumper job exists in the linked jobs file.
- The new ProwJob CRD URL in the `tackle` instructions resolves.
- The named autobumper postsubmit deploys Terraform resources; the author disclosed this example as a reviewer choice in the PR description.
- The linked pushgateway file contains commented-out manifests; the page's surrounding deployment description predates this link repair.
- No automated site build or link-check suite was run; the remote targets above were inspected directly.

## Open questions

- Should the `generic-autobumper.md` example replace its `test-infra` `upstreamURLBase` now that `test-infra` no longer carries Prow deployment configuration?
- Would a postsubmit that deploys Prow manifests be a clearer example than `post-k8sio-deploy-prow-build-trusted-resources` at `site/content/en/docs/components/cli-tools/generic-autobumper.md:25`?
- Should `site/content/en/docs/metrics/_index.md:72` describe the linked pushgateway file as a disabled template rather than an active deployment?

## Followups

### 1. Refresh the generic-autobumper guide

- category: docs
- where: `site/content/en/docs/components/cli-tools/generic-autobumper.md:8-88`
- necessity: should — the repository and job examples can mislead readers copying the setup.
- why followup: PR #954 repaired dead links; the author explicitly left the choice of postsubmit example to reviewers, and the wider sample still assumes Prow deployment config lives in `kubernetes/test-infra`.
- handoff prompt:

```text
In kubernetes-sigs/prow, following PR #954 ("docs: repair the dead links in the site content"), refresh site/content/en/docs/components/cli-tools/generic-autobumper.md for the current Prow deployment layout. Work from the default branch after PR #954 lands. Check the current manifest repository and the actual autobumper and deployment jobs before editing. Make the sample values for extra_refs, gitHubOrg/gitHubRepo, remoteName, upstreamURLBase, includedConfigPaths, refConfigFile, and stagingRefConfigFile internally consistent with a real repository, or clearly mark historical examples as such. Replace or reword the postsubmit example at line 25 so it does not present a Terraform resource job as a manifest deployment job. Verify that every concrete repository path, job name, and URL you leave in the guide exists and matches the surrounding claim. Change documentation only; do not modify Prow code, job definitions, or cluster configuration.
```

### 2. Reconcile the Pushgateway documentation with the current deployment

- category: docs
- where: `site/content/en/docs/metrics/_index.md:65-72`
- necessity: could — the linked file is reachable but contains only commented-out resources.
- why followup: The link repair made the target reachable, but determining whether the surrounding deployment description is current requires a separate architecture check.
- handoff prompt:

```text
In kubernetes-sigs/prow, following PR #954 ("docs: repair the dead links in the site content"), reconcile site/content/en/docs/metrics/_index.md:65-72 with the current metrics deployment. Work from the default branch after PR #954 lands. The linked kubernetes/k8s.io kubernetes/gke-prow/prow/pushgateway.yaml currently contains only commented-out resources, while Prow starter manifests still reference --push-gateway. Inspect the current deployment source and relevant metrics configuration to establish whether Pushgateway and its proxy are active, historical, or replaced. Update the section and its link so readers are told what is actually deployed; if only a disabled template exists, say so instead of presenting it as an active manifest. Verify the factual claims and target contents, not only HTTP status. Change documentation only; leave runtime flags, manifests, and monitoring configuration outside this task.
```

### 3. Triage the remaining external-link backlog

- category: docs
- where: `site/content/en/docs/overview/_index.md`, `private-deck.md`, `metadata-artifacts.md`, and `announcements.md`
- necessity: could — the remaining dead ends are reader-facing but need decisions beyond a direct URL replacement.
- why followup: PR #954 deliberately left these links untouched because their correct replacements or access policy were unclear. The two `generic-autobumper.md` examples belong to followup 1 and are excluded here.
- handoff prompt:

```text
In kubernetes-sigs/prow, following PR #954 ("docs: repair the dead links in the site content"), triage the remaining external links listed under "Links that need a maintainer decision" in that PR's description, excluding both generic-autobumper.md entries covered by the guide followup. Work from the default branch after PR #954 lands. Check the adopter URLs for Jetstack and KubeSphere in site/content/en/docs/overview/_index.md; the two oss-prow.knative.dev examples in private-deck.md; the started.json, finished.json, and expired GCS demo links in metadata-artifacts.md; and the expired GCS demo in announcements.md. Verify each target's present behavior, find a correct replacement where one exists, and update or remove reader-facing links that no longer serve their purpose. Where a URL is intentionally access-controlled, make that limitation clear in the documentation. Report any externally controlled failures that have no safe documentation fix. Acceptance: every edited link supports its surrounding claim, and the eight scoped URLs each have an explicit disposition. Keep the work to site documentation; do not change credentials, storage policy, external services, or the link-checker implementation in PR #952.
```
