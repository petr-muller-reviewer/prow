---
issue: kubernetes-sigs/prow#587
title: "prow v1 release pre-reqs"
state: closed
labels:
main_sha: 5765df26189ddc41cb9c975e89d613f6c26282ef
triaged_at: 2026-08-31T22:52:09Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 4
recommended_labels: [kind/feature, help wanted, priority/backlog]
---

## Verdict

Accepted: legitimate, still-unresolved maintainer roadmap issue that expired via stale-bot inactivity (not maintainer decision). Recommend reopening after confirming the goal is still active, then splitting into independently-closeable sub-issues rather than re-triaging as one omnibus checklist.

## What the issue reports

- Maintainer (upodroid) proposes prerequisites for Prow's first tagged release (v0.1.0/v1).
- Seven items: create `prow-announcements@kubernetes.io`; tag+release main as v0.1.0; resolve #585; make any other desired breaking changes; convert `prowjobs.js` to a versioned API served like `deck/v1/` with kubernetes semantics; dependency cleanup; deprecate unneeded components.
- Also describes the intended release mechanics: cut a release in this repo, promote images to `registry.k8s.io/prow/[COMPONENTS]`.

## Findings

### [cause] Auto-closed by inactivity, not resolution
- detail: Closed 2026-06-07 by `k8s-triage-robot` as "not-planned" purely from the 90d(stale)+30d(rotten)+30d(close) lifecycle. Only prior activity was one unfollowed `/assign` comment (rajibmitra, 2026-01-08). No maintainer discussion ever occurred, and none of the 7 prerequisites were resolved or rejected.
- evidence: issue comments timeline, `gh issue view 587 --repo kubernetes-sigs/prow --json comments`.

### [cause] Omnibus checklist structure prevented closure
- detail: Bundling 7 independent workstreams into one issue meant no single unit of progress could close it, so it never accumulated visible activity and fell to the stale-bot.
- evidence: n/a (structural observation from issue body + lack of any linked sub-issue or PR).

### [related-code] Deck's prowjobs API is unversioned (target of prereq #5)
- where: `prow/cmd/deck/main.go:457` (`handleProwJobs`, serves `/prowjobs.js`), `main.go:456` (`/data.js`), `main.go:528` (singular `/prowjob`), `main.go:315-316` (`/config`, `/plugin-config`)
- excerpt: |
    No `/v1/`-prefixed path exists anywhere in the repo (searched cmd/deck, pkg/gangway). `pkg/gangway/gangway.proto` (gRPC/REST job-trigger gateway) also has no version segment.

### [related-code] Partial release-build plumbing exists (prereq #2)
- where: `.ko.yaml` (repo root, goreleaser-format, ~35 build entries), `cloudbuild.yaml`, `.prow-images.yaml`
- excerpt: |
    ldflags -X sigs.k8s.io/prow/pkg/version.Version={{.Env.VERSION}}
- detail: Drives per-commit image pushes to `us-docker.pkg.dev/k8s-infra-prow/images`, not a tagged release. No `.github/workflows/` exist (only `.github/dependabot.yaml`). `git tag` returns zero tags; `gh release list` returns none.

### [related-code] RELEASE.md is unmodified template boilerplate
- where: `prow/RELEASE.md`
- detail: Still references the generic "Kubernetes Template Project" process and `kubernetes-dev@googlegroups.com`, not `prow-announcements@kubernetes.io` or any Prow-specific release process.

### [related-issue] kubernetes-sigs/prow#585
- ref: kubernetes-sigs/prow#585
- relevance: "Remove unneeded prow plugins/cmds" — explicit blocker for prerequisite #3, confirmed still **open** with no recent activity.

### [related-pr] Incremental plugin/cmd reorg (partial progress on #3/#7)
- ref: commits `feeddb49e` (delete `cmd/grandmatriarch`), `ccdbac4ea` (extract `rifle` plugin from `blunderbuss`), `76112a601` (split `review_assignment` into its own package)
- relevance: Reorganization, not net reduction — 37 dirs remain under `cmd/`, 62 under `pkg/plugins/`.

### [related-pr] Isolated dependency cleanup (partial progress on #6)
- ref: commit `7953d2b1d` ("fix(deck): replace gorilla/csrf with net/http.CrossOriginProtection")
- relevance: Only deliberate manual dependency-reduction commit found; `go.mod` still has 229 `require` entries with no larger pruning effort visible.

## Checked

- `gh release list --repo kubernetes-sigs/prow` — no releases exist; `git tag` — no tags exist.
- Issue #585 status — confirmed still open via `gh issue view`.
- Cross-referenced PR `openshift/release#81376` ("Add z-stream testing") — unrelated downstream consumer PR, not relevant.
- Full comment/timeline history on #587 — only bot lifecycle comments plus one unfollowed `/assign`.
- Searched repo for `/v1/`-style versioned API paths (`cmd/deck`, `pkg/gangway`) — none found.
- `git log -- go.mod` — dominated by routine dependabot bumps.

## Next steps

- Confirm with SIG Testing / Prow maintainers whether the v0.1.0 release effort is still desired before reopening.
- If still desired: reopen #587, clear `lifecycle/rotten`, and split into per-prerequisite tracking issues (release tagging/automation + RELEASE.md + mailing list; #585 cleanup; dependency cleanup; versioned-API design) so each can progress and close independently.
- Comment on #585 to unblock prerequisite #3 — the one concrete, still-open, actionable sub-item with no current activity.
- Apply labels if reopened: `kind/feature`, `help wanted` (on split sub-issues), `priority/backlog`.

## Open questions

- Is a Prow v1 release still an active goal for maintainers, or has planning moved elsewhere (a KEP, a different tracking issue, informal SIG consensus)?
- Should this be split into separate, independently-triageable issues per prerequisite rather than tracked as one omnibus checklist?
- What does "kubernetes semantics" mean concretely for the versioned prowjobs API (pagination? watch support? which verbs?) — needs a design proposal before implementation.
