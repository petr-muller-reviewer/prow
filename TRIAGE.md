---
issue: kubernetes-sigs/prow#1021
title: "pull-prow-integration: gangway's fixed nodePort 32000 can collide with an automatically assigned NodePort"
state: open
labels: kind/flake
main_sha: 4425e386ffba859f8fbfb535617559f05d37bf2f
triaged_at: 2026-10-19T14:09:44Z
verdict: accepted
advice:
  advised_at: 2026-10-20T12:31:10Z
  based_on_triaged_at: 2026-10-19T14:09:44Z
---

# Triage

## Verdict

**accepted** — This is a reproducible failure in Prow's integration setup: Kubernetes can allocate NodePort 32000 to an earlier Service, then reject gangway's fixed claim before tests start. Keep the issue open while PR #1022, which addresses the collision, is reviewed; its integration presubmit passes.

## What the issue reports

- `pull-prow-integration` failed while applying gangway's Service with `provided port is already allocated`; the run produced no junit file because tests did not start.
- Gangway pins NodePort 32000, while earlier NodePort Services use automatic allocation and fakepubsub also receives an explicit selected port.
- The report proposes moving gangway to the lower static band and keeping fakepubsub's selection out of that port.

## Findings

### [reproducibility] reported CI failure before tests
- detail: The linked run failed applying gangway because NodePort 32000 was already allocated. The issue reports no `junit-integration.xml`; this triage did not independently rerun the failure.
- evidence: [Prow build for PR #1013](https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_prow/1013/pull-prow-integration/2111481444873277440)

### [cause] gangway claims a port after automatic NodePorts are created
- detail: Earlier Services can receive 32000 automatically, then gangway's explicit claim fails. In the default range, Kubernetes prefers dynamic allocation from 30086-32767, which includes 32000; the static band is 30000-30085.
- evidence: `test/integration/lib.sh:201-255`; [Kubernetes NodePort allocation docs](https://kubernetes.io/docs/concepts/services-networking/service/#reserve-nodeport-ranges-to-avoid-collisions)

### [related-code] gangway fixes NodePort 32000
- where: `test/integration/config/prow/cluster/gangway_service.yaml:10-18`
- excerpt: |
      ports:
      - name: grpc
        port: 32000
        targetPort: 32000
        nodePort: 32000
        protocol: TCP

### [related-code] automatic NodePort Services precede gangway
- where: `test/integration/lib.sh:201-255`
- excerpt: |
      hook_rbac.yaml
      hook_service.yaml
      hook_deployment.yaml
      ...
      deck_rbac.yaml
      deck_service.yaml
      deck_deployment.yaml
      deck_tenant_deployment.yaml
      ...
      gangway_rbac.yaml
      gangway_service.yaml
      gangway_deployment.yaml

### [related-code] fakepubsub gets an explicit port and kind forwards gangway traffic
- where: `test/integration/integration-test.sh:198-207`, `test/integration/lib.sh:504-508`, `test/integration/setup-kind-cluster.sh:176-187`, `test/integration/test/gangway_test.go:158-159`
- excerpt: |
      fakepubsub_node_port="$(get_random_node_port)"
      ...
      function get_random_node_port() {
        # 30000-32767 is the default NodePort range. If "shuf" isn't available, use
        # 30303 as a default.
        shuf -i 30000-32767 -n 1 || echo 30303
      }
      ...
        containerPort: 32000
        hostPort: 32000
      ...
      c, err := gangwayGoogleClient.NewInsecure(":32000", tt.projectNumber)

### [related-pr] PR #1022 moves gangway into the static band
- ref: kubernetes-sigs/prow#1022
- relevance: The open PR moves gangway to NodePort 30000, maps host port 32000 to container port 30000, and selects fakepubsub ports from 30001-30085. Its `pull-prow-integration` check passes; existing kind clusters need the updated mapping or recreation.

## Resolved

None.

## Checked

- Read issue #1021 and its timeline; PR #1022 is the only cross-reference. Similar issue searches for `gangway nodePort`, `nodePort collision`, and `pull-prow-integration gangway` found no other matching issue.
- Read NodePort setup order, gangway and fakepubsub manifests, integration test connection, and kind port mapping.
- Confirmed issue #1022 remains open and its `pull-prow-integration` check passes; the Tide check is pending approval and `lgtm` labels.
- Confirmed the default static/dynamic NodePort bands in Kubernetes documentation. No independent local test run was performed.

## Next steps

- Review PR #1022's static-band allocation, fakepubsub range, and kind mapping/migration note.
- Keep #1021 open until the PR is merged and CI confirms the failure is resolved; then close it as fixed.
- Consider adding `area/gangway` while retaining the existing `kind/flake` label.

## Advice

No issue activity since triage: #1021 remains open with `kind/flake`, no new comments were posted, and PR #1022 remains the only linked PR. The existing next steps still apply.

1. **Review PR #1022 before pursuing separate work.** The PR is open and mergeable, its `pull-prow-integration` check passes, and Tide is waiting for the `approved` label. The issue is already cross-referenced by the PR.

   ```sh
   gh pr view 1022 --repo kubernetes-sigs/prow
   gh pr diff 1022 --repo kubernetes-sigs/prow
   ```

2. **Optionally add `area/gangway`.** That label exists and is not currently applied; keep `kind/flake`.

   ```sh
   gh issue edit 1021 --repo kubernetes-sigs/prow --add-label area/gangway
   ```

Keep #1021 open until PR #1022 is merged and CI confirms the failure is resolved. No milestone or help-wanted label is recommended while the fix is already in flight.

## Open questions

None identified during triage.
