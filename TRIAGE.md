---
issue: kubernetes-sigs/prow#559
title: "Update Base Prow Images"
state: open
labels: lifecycle/rotten
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:32:02Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [help-wanted]
---

## Findings

### [reproducibility] base image staleness has already recurred once since filing
- detail: The issue reported base images stale since July 2025. Since filing (2025-12-01), `.ko.yaml` has been manually bumped twice — confirming the staleness recurs whenever nobody automates it.
- evidence: `git log --oneline -- .ko.yaml` shows `da4351c35` ("Update base container images to v20251206-8481528a19", 2025-12-15) and `6312f82b3` ("Update base container images to v20260512-1f34ef0df3", 2026-06-09). Both are human-authored commits, not bot commits.

### [cause] no periodic ProwJob invokes generic-autobumper for this repo
- detail: PR #796 adds `hack/autobump.yaml`, a `generic-autobumper` config, but that config is inert without a periodic job invoking `generic-autobumper --config=hack/autobump.yaml`. Prow job definitions for `kubernetes-sigs/prow` live in a different repository (`kubernetes/test-infra`), and a search of `config/jobs/kubernetes-sigs/prow/{prow-periodics,prow-presubmits,prow-postsubmits}.yaml` there found no such job.
- evidence: `gh api search/code -f q='repo:kubernetes/test-infra path:config/jobs/kubernetes-sigs/prow'` → only `prow-periodics.yaml`, `prow-presubmits.yaml`, `prow-postsubmits.yaml`, none referencing `generic-autobumper` or `hack/autobump.yaml`.

### [related-code] base image pins
- where: `.ko.yaml:7-56`
- excerpt: |
    baseImageOverrides:
      sigs.k8s.io/prow/cmd/branchprotector: gcr.io/k8s-staging-test-infra/alpine:v20260512-1f34ef0df3
      sigs.k8s.io/prow/cmd/checkconfig: gcr.io/k8s-staging-test-infra/git:v20260512-1f34ef0df3
      ...

### [related-code] generic-autobumper config added by PR #796 (unmerged)
- where: `hack/autobump.yaml` (new file, not yet on main)
- excerpt: |
    gitHubLogin: "k8s-infra-ci-robot"
    gitHubToken: "/etc/github-token/token"
    gitHubOrg: "kubernetes-sigs"
    gitHubRepo: "prow"
    targetVersion: "latest"
    extraFiles:
      - ".ko.yaml"
    prefixes:
      - name: "k8s-staging-test-infra GCR images"
        prefix: "gcr.io/k8s-staging-test-infra"
        repo: "https://github.com/kubernetes/test-infra"

### [related-code] template periodic job pattern for the missing piece
- where: `kubernetes/test-infra:config/jobs/kubernetes/sig-k8s-infra/trusted/sig-k8s-infra-test-infra.yaml`
- excerpt: |
    image: us-docker.pkg.dev/k8s-infra-prow/images/generic-autobumper:v20260714-0633879af
    command: [generic-autobumper]
    args: [--config=config/prow/autobump-config/prow-job-autobump-config.yaml]
    volumeMounts:
    - name: github
      mountPath: /etc/github-token
      readOnly: true
    volumes:
    - name: github
      secret:
        secretName: k8s-infra-ci-robot-github-token

### [related-pr] manual bump #1
- ref: kubernetes-sigs/prow#567
- relevance: "Update base images", merged 2025-12-22, first manual response to this issue.

### [related-pr] manual bump #2
- ref: kubernetes-sigs/prow#747
- relevance: "Update base container images to v20260512-1f34ef0df3", merged 2026-06-10; shows staleness recurs without automation.

### [related-pr] the actual fix in flight
- ref: kubernetes-sigs/prow#796
- relevance: "autobump prow base images", opened 2026-07-09 by upodroid, body says `Fixes: #559`. Adds `hack/autobump.yaml`. Still OPEN, mergeable, no reviews yet as of this triage.

### [related-issue] dependabot prerequisite
- ref: kubernetes/test-infra#35799
- relevance: "enable dependabot bumping of docker images", merged 2025-12-04; upstream prerequisite discussed in this issue's thread.

### [related-issue] related base-image vulnerability fix
- ref: kubernetes/test-infra#35968
- relevance: "Update base images and dependencies to address vulnerabilities", merged 2025-12-05; also referenced in the thread.

## Checked

- Current `.ko.yaml` pins (`v20260512-1f34ef0df3`) — not stale as of this triage; original symptom already patched manually twice.
- `kubernetes/test-infra` job configs under `config/jobs/kubernetes-sigs/prow/` for an existing autobump periodic — none found.
- `generic-autobumper` usage across `kubernetes/test-infra` for a template pattern to copy — found `sig-k8s-infra-test-infra.yaml`.
- PR #796 diff, review state, and CI status — 1 file / 19 lines, mergeable, no reviews, no failing checks.
- This repo for GitHub Actions/dependabot config of its own — none; `kubernetes-sigs/prow` uses Prow itself for CI, not GitHub Actions.

## Next steps

- Get PR #796 reviewed and merged.
- File (or find a volunteer for) a companion periodic-job PR in `kubernetes/test-infra` (`config/jobs/kubernetes-sigs/prow/prow-periodics.yaml`), modeled on `config/jobs/kubernetes/sig-k8s-infra/trusted/sig-k8s-infra-test-infra.yaml`.
- Apply `/help-wanted`; run `/remove-lifecycle rotten` once active work resumes.
- Consider a separate tracking issue for BenTheElder's proposal to move Prow-only base images into this repo (larger, Level 3+, out of scope for closing #559).

## Open questions

- Does the missing periodic job belong to sig-testing (test-infra owners) or can prow maintainers self-serve it in `kubernetes/test-infra`'s trusted cluster?
- Should #796 be merged before or together with the companion test-infra periodic job, given the config is inert on its own?
