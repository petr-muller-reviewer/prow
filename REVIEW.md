---
pr: kubernetes-sigs/prow#1036
title: "Update k8s-staging-test-infra GCR images as needed"
head_sha: 7dddfa689c93b802fd2983e431a084d61f56b800
base: main
reviewed_at: 2026-10-27T17:17:33Z
verdict: approve
---

# Review

## Verdict

Approve. A complete and internally consistent autobump: all 38 `.ko.yaml` base-image
references move to tags that exist in the registry and were published from the expected
test-infra commits, each command keeps its previous image family, and the only upstream
content change in the window is the Alpine base bump to 3.24.3. The PR is already merged;
`upstream/main` still carries exactly this content.

## What this PR does

- Bumps every `baseImageOverrides` entry in `.ko.yaml` from the 2026-05-12 build
  (`v20260512-1f34ef0df3`) to the 2026-10-23 builds.
- Alpine-based components move to `alpine:v20261023-49052c49ef` (18 entries).
- Git-based components move to `git:v20261023-d6356df2e8` (14 entries).
- Components that talk to Kubernetes move to
  `git-custom-k8s-auth:v20261023-d6356df2e8` (6 entries).
- No command's image family changes, and the distroless / `fakepubsub` overrides are left
  untouched; upstream, all three Dockerfiles only move `FROM alpine:3.23.4` to
  `alpine:3.24.3`.

## Findings

None (no blocking, should-fix, or nit findings).

## Checked

- All 38 GCR entries were updated; no `v20260512-1f34ef0df3` / `1f34ef0df3` remains
  anywhere in the tree (including hidden files; `.prow-images.yaml` only lists build
  directories, no tags).
- Tag/family pairing is uniform: 18x `alpine` -> `49052c49ef`, 14x `git` and 6x
  `git-custom-k8s-auth` -> `d6356df2e8`. No line mixes a tag from one image family into
  another.
- All three new tags exist under `gcr.io/k8s-staging-test-infra` and were pushed on
  2026-10-23 (alpine 20:04Z, git 20:54Z, git-custom-k8s-auth 21:00Z), after the matching
  test-infra merges (`49052c49ef` 20:02Z, `d6356df2e8` 20:45Z).
- Upstream delta between the old and new builds is only `FROM alpine:3.23.4` ->
  `alpine:3.24.3` in `images/{alpine,git,git-custom-k8s-auth}/Dockerfile`; no other
  Dockerfile changes. The five-month jump is because the autobumper was broken until
  #1034 landed just before this PR, not because of additional image changes.
- Multi-arch coverage is preserved (amd64, arm64, ppc64le, s390x, riscv64) for all three
  images. The media type changed from Docker manifest lists to OCI image indexes; both are
  handled by `ko`/containerd, and `pull-prow-image-build-test` and `pull-prow-integration`
  passed.
- `.ko.yaml` parses as YAML with 40 overrides (38 bumped + distroless + `fakepubsub`);
  the key set is unchanged from the base commit.
- `hack/autobump.yaml` sets `consistentImages: false`, so the differing per-image suffixes
  are expected. Other `k8s-staging-test-infra` references are `:latest` dev/integration
  usages and correctly not pinned.
- `upstream/main` still has identical content; no post-merge fix or revert.

## Open questions

- None.

## Followups

Accepted 4 of 4 opportunities on 2026-10-27 (post-merge followup to #1036, merge commit `0d9737ca26be70c30ad4d017f38965d0f8ef0943`).

### docs: `.ko.yaml` base-image header is stale
- category: docs
- necessity: could
- where: `.ko.yaml:1-5`

```text
In kubernetes-sigs/prow, following post-merge followup to PR #1036 ("Update k8s-staging-test-infra GCR images as needed", merged as 0d9737ca26be70c30ad4d017f38965d0f8ef0943):

Rewrite the header comment at the top of `.ko.yaml` (lines 1-5). It currently says "Prow uses 2 bases images", omits the plain `git` image entirely, has a missing word in "which the same alpine image", and has a trailing space on the third line.

Task:
- Document all three base images used by `baseImageOverrides`: `alpine`, `git`, and `git-custom-k8s-auth`.
- For each, state what it contains and when to use it. Derive the exact contents from `images/alpine/Dockerfile`, `images/git/Dockerfile`, and `images/git-custom-k8s-auth/Dockerfile` in kubernetes/test-infra (branch master), not from memory: roughly, `alpine` is Alpine plus Prow's required runtime packages; `git` is the alpine image plus git; `git-custom-k8s-auth` is the git image plus AWS/GCP Kubernetes auth clients.
- Keep the existing note about the distroless default base image and the rule that `git-custom-k8s-auth` is only for components that talk to Kubernetes clusters.
- Fix the grammar and remove trailing whitespace.

Acceptance criteria:
- The header names all three images, each with an accurate one-line description and usage rule.
- No line has trailing whitespace, and `.ko.yaml` still parses as YAML.
- The `baseImageOverrides` entries are byte-for-byte unchanged.

Scope guard:
- Comment-only change. Do not touch image tags, mappings, or any other file.
```

### tech-debt: guard against images silently missing a base-image override
- category: tech-debt
- necessity: could
- where: `hack/tools/prowimagebuilder/main.go`, `hack/tools/prowimagebuilder/main_test.go`

```text
In kubernetes-sigs/prow, following post-merge followup to PR #1036 ("Update k8s-staging-test-infra GCR images as needed", merged as 0d9737ca26be70c30ad4d017f38965d0f8ef0943):

Add validation to `hack/tools/prowimagebuilder` so a Prow image cannot silently build on the default distroless base because it was added to `.prow-images.yaml` but not to `.ko.yaml`.

Task:
- In `hack/tools/prowimagebuilder/main.go`, when the images file is loaded, parse `path.Join(rootDir, ".ko.yaml")` (the tool already imports `sigs.k8s.io/yaml`; `rootDir` comes from `git rev-parse --show-toplevel`).
- Assert that every `dir` in `.prow-images.yaml` has a matching `sigs.k8s.io/prow/<dir>` key under `baseImageOverrides`. Fail with an error that lists every missing dir.
- Only enforce this direction: `.ko.yaml` legitimately contains extra entries for integration-test images (`test/integration/cmd/fake*`) that are not in `.prow-images.yaml`.
- Add table-driven tests in `hack/tools/prowimagebuilder/main_test.go` covering all-present and missing-entry cases, using fixture or temp files rather than the real repo files.

Acceptance criteria:
- `cd hack/tools && go test ./prowimagebuilder/...` passes.
- Invoking the validation against the current `.prow-images.yaml` and `.ko.yaml` succeeds; a fixture with a dir absent from `.ko.yaml` produces a clear error naming it.
- No network calls and no new CI job.

Scope guard:
- Do not edit `.prow-images.yaml` or `.ko.yaml` contents. Do not change the image build logic or add platform/architecture checks.
```

### tech-debt: partial base-image bumps fail silently
- category: tech-debt
- necessity: could
- where: `cmd/generic-autobumper/imagebumper/imagebumper.go:227-275`, `cmd/generic-autobumper/main.go:360-414`

```text
In kubernetes-sigs/prow, following post-merge followup to PR #1036 ("Update k8s-staging-test-infra GCR images as needed", merged as 0d9737ca26be70c30ad4d017f38965d0f8ef0943):

Stop generic-autobumper from silently producing a partial base-image bump.

Current behavior: `updateAllTags` in `cmd/generic-autobumper/imagebumper/imagebumper.go` (~lines 227-259) logs `tagPicker` errors with `log.Printf`, keeps the old tag, and continues; `UpdateFile` (~lines 262-275) then returns nil, so `updateReferences` in `cmd/generic-autobumper/main.go` (~lines 360-414) reports success. Because `hack/autobump.yaml` sets `consistentImages: false`, `getVersionsAndCheckConsistency` cannot detect the stale image. A bump PR can therefore ship with only some of `alpine`/`git`/`git-custom-k8s-auth` updated.

Task:
- Make `updateAllTags` collect each tag-lookup failure (host, image, current tag, and underlying error) instead of only logging it.
- Propagate a non-nil error out of `UpdateFile` (and therefore `updateReferences`) when any lookup failed, after still writing the successful replacements. The error message must name every image that failed.
- Update the existing test case "file with an error image is still otherwise updated" in `cmd/generic-autobumper/imagebumper/imagebumper_test.go` to assert the new error behavior, and add a case asserting the error text names the failing image.
- Keep the successful-path behavior identical (all images bumped, no error).

Acceptance criteria:
- `go test ./cmd/generic-autobumper/...` passes.
- A failing `tagPicker` for one image yields a non-nil error naming that image, while the other images are still replaced in the written content.
- `ci-prow-autobump-base-images` in kubernetes/test-infra will now exit non-zero on such a partial bump, which its existing testgrid alert covers after repeated failures.

Scope guard:
- Do not change tag-picking rules, config schema, or `hack/autobump.yaml`.
- If maintainers prefer best-effort bumps, the alternative is to surface failed images in the PR body instead of failing the run; raise that as a question rather than silently changing behavior.
```

### docs: the new `--dry-run` flag is undocumented
- category: docs
- necessity: could
- where: `site/content/en/docs/components/cli-tools/generic-autobumper.md`

```text
In kubernetes-sigs/prow, following post-merge followup to PR #1036 ("Update k8s-staging-test-infra GCR images as needed", merged as 0d9737ca26be70c30ad4d017f38965d0f8ef0943):

Document the `--dry-run` flag of generic-autobumper, added in #1034 but absent from the docs.

Task:
- Add a short "Dry run" section to `site/content/en/docs/components/cli-tools/generic-autobumper.md` (after the workflow bullets, before "### Requirement").
- Describe the behavior from `cmd/generic-autobumper/bumper/bumper.go` (`processDryRun`) and `cmd/generic-autobumper/main.go:190`: apply the changes to the working tree, log the generated PR title and body, and skip commit/push/PR creation; read-only lookups (upstream versions, on-call) still happen; token/App/Gerrit-cookie validation is waived for fork/branch/Gerrit modes.
- Include one concrete local example for validating a base-image bump, e.g. from the repo root: `go run ./cmd/generic-autobumper --config=hack/autobump.yaml --dry-run`, then inspect `.ko.yaml` with `git diff`. Verify the command and flag name against the code before documenting them.

Acceptance criteria:
- The page has a "Dry run" section with the behavior above and the example.
- The example is consistent with the actual flags (`--dry-run`, `--config`, `--skip-pullrequest`).
- No other docs page needs changes for the flag.

Scope guard:
- Docs only. Do not change flag behavior, validation, or `hack/autobump.yaml`.
```
