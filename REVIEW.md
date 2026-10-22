---
pr: kubernetes-sigs/prow#1022
title: "integration: move gangway's NodePort into the static band"
head_sha: 08363f55b0395c6a04f19069b7ad638947915a9a
base: main
reviewed_at: 2026-10-21T23:15:46Z
verdict: approve
refresh_log:
  - old_sha: 08363f55b0395c6a04f19069b7ad638947915a9a
    new_sha: 08363f55b0395c6a04f19069b7ad638947915a9a
    summary: "No code changes; recorded Prucek's approval and the approval notifier comment."
gate:
  decision: hold
  gated_at: 2026-10-19T12:49:05Z
  gated_head_sha: 08363f55b0395c6a04f19069b7ad638947915a9a
  reviewed_head_sha: 08363f55b0395c6a04f19069b7ad638947915a9a
---
# Review

## Gate

**Verdict: HOLD**

The existing-cluster migration remains a should-fix finding. The PR description gives users the teardown and custom-config steps, but the durable setup guide does not; setup reuses clusters with the stale mapping. Hold until the guide documents the one-time migration, or the maintainer explicitly accepts this documentation gap.

### Gating list

- REVIEW.md should-fix finding at site/content/en/docs/local-dev-two-machines.md:160-161 remains unaddressed. Add the teardown command and custom Kind mapping update.

### Area 2 — Independent merge risk

There are no production-facing API or configuration changes. Existing developer/test Kind clusters retain the old host-port mapping because setup-kind-cluster.sh reuses running clusters (lines 120-125) and only applies the new container-port mapping when creating a cluster (lines 183-191). This can leave Gangway unreachable to local integration clients until users recreate the cluster. The PR description documents the manual migration, so the risk is limited to contributors who miss that note or use a custom Kind config.

## Verdict

Approve. The change moves Gangway’s fixed NodePort into the static band and keeps the existing host port used by integration clients. Code Quality found no correctness issues; Maintainability and Deployment Risk independently recommend documenting the one-time migration for existing Kind clusters.

## What this PR does

- Assigns Gangway NodePort 30000 while retaining Service and container port 32000.
- Maps Kind container port 30000 to host port 32000 for the integration cluster.
- Updates the two-machine development Kind example to use the same mapping.
- Restricts CI fakepubsub random NodePorts to 30001–30085.

Since previous review:
- No code changes; the PR head remains `08363f55b0395c6a04f19069b7ad638947915a9a`.
- Prucek submitted an `APPROVED` review at 2026-10-19 20:05:18 UTC. `kubernetes-prow[bot]` posted an approval workflow comment at 20:05:25 UTC, noting that approval from `droslean` is still required; the PR remains open and currently has the `lgtm` label.

## Findings

### [should-fix] Document the existing-cluster migration
- where: `site/content/en/docs/local-dev-two-machines.md:160-161`
- concern: The PR description explains that existing Kind clusters need recreation and custom configs need the new container port, but the durable setup guide only shows the new mapping. Add the one-time teardown and custom-config instructions so developers using a reused cluster know how to restore Gangway connectivity.
- excerpt: |
    - containerPort: 30000 # gangway's NodePort
      hostPort: 32000 # flexible based on your machine

## Checked

- Issue #1021’s collision scenario and requested static-band allocation.
- Gangway Service and Deployment ports, integration client connections on port 32000, and Kind host-to-container port mappings.
- fakepubsub’s randomized port selection excludes Gangway’s 30000.
- setup-kind-cluster.sh reuses a running cluster, so existing clusters retain their old port mapping until recreated.
- No production-facing configuration changes were identified.
- git diff --check passed. The integration suite was not run during this review.

## Open questions

None.
