---
pr: kubernetes-sigs/prow#796
title: "autobump prow base images"
head_sha: ab99df655a530e6e7b30a29efad116423e4d8018
base: main
reviewed_at: 2026-11-07T23:54:40Z
verdict: request-changes
refresh_log:
  - at: 2026-11-07T23:54:40Z
    old_sha: ab99df655a530e6e7b30a29efad116423e4d8018
    new_sha: ab99df655a530e6e7b30a29efad116423e4d8018
    summary: "No PR code changes; PR merged 2026-08-10 after approval. Post-merge upstream commit 7b32f3d6f relaxed generic-autobumper validation to accept extraFiles alone, resolving the blocking finding; scheduled job then produced merged bot PR #1036 bumping .ko.yaml. Both open questions answered."
---

## What this PR does

- Adds `hack/autobump.yaml`, a new config for the in-tree `cmd/generic-autobumper` tool.
- Targets images under the `gcr.io/k8s-staging-test-infra` prefix, referenced as base images in `.ko.yaml`, via `extraFiles: [".ko.yaml"]`.
- Fixes #559: those images now expire after 6 months and `.ko.yaml` hadn't been bumped since ~July 2025.
- Purely additive: one new file, not referenced by any existing build/test/deploy path.

Since previous review (`ab99df65`, unchanged through merge):
- PR merged 2026-08-10T19:58:05Z as merge commit `85e45f6d2` with an approval from matthyx; `hack/autobump.yaml` landed exactly as reviewed.
- Post-merge, `cmd/generic-autobumper/main.go` commit `7b32f3d6f` (2026-10-23) relaxed validation so `extraFiles` alone is sufficient, which resolves the blocking finding.
- The scheduled job in `kubernetes/test-infra` then produced bot PRs #1035 (closed) and #1036 (merged 2026-10-23), touching only `.ko.yaml` — the config now works end to end.

## Findings

No open findings.

## Resolved

### [blocking] missing `includedConfigPaths` causes the bumper to fail on every invocation — resolved upstream
- where: `hack/autobump.yaml` (whole file); was validated against `cmd/generic-autobumper/main.go:123,232-234`
- concern (at review time): `IncludedConfigPaths []string \`yaml:"includedConfigPaths"\`` was unconditionally required by `validateOptions`, which returned `errors.New("includedConfigPaths is mandatory")` when empty. `main()` calls `validateOptions` before any bump logic runs and `logrus.Fatalf`s on error, so `generic-autobumper --config=hack/autobump.yaml` exited nonzero immediately and never reached `.ko.yaml` via `extraFiles`.
- resolved by: `cmd/generic-autobumper/main.go`, commit `7b32f3d6f` on `main` (2026-10-23, "add dry-run flag to bumper and fix an error in the bumper"), which changed the check to `if len(o.IncludedConfigPaths) == 0 && len(o.ExtraFiles) == 0` with the message `at least one of includedConfigPaths or extraFiles must be specified`. `hack/autobump.yaml` sets `extraFiles: [".ko.yaml"]`, so it now validates.
- verified live: scheduled run produced `k8s-infra-ci-robot` PR #1035 (closed) and #1036 (merged 2026-10-23T21:09Z); both touched only `.ko.yaml` and #1036 bumped `alpine`, `git`, `git-custom-k8s-auth` from the 2026-05-12 staging images to 2026-10-23.
- caveat: the config as merged (2026-08-10) was broken until `7b32f3d6f` (2026-10-23), so scheduled runs in that window presumably failed and #559 was not actually fixed for ~2.5 months.

### [question] no periodic job wired to run this config — answered
- where: `hack/autobump.yaml` (whole file)
- resolved: `kubernetes/test-infra` file `config/jobs/kubernetes/sig-k8s-infra/trusted/sig-k8s-infra-prow.yaml` defines job `ci-prow-autobump-base-images` — cron `15 14-20/5 * * 1-5`, cluster `k8s-infra-prow-build-trusted`, `extra_refs` `kubernetes-sigs/prow@main`, running `generic-autobumper --config=hack/autobump.yaml --labels-override=skip-review`, mounting secret `k8s-infra-ci-robot-github-token` at `/etc/github-token` (matches `gitHubToken: "/etc/github-token/token"`).
- note: the job already existed before the review (present at test-infra ref `2ceed8557`, 2026-07-20); the original review only searched this repo, so it missed the external job config.

## Checked
- All other fields (`gitHubLogin`, `gitHubToken`, `gitName`, `gitEmail`, `skipPullRequest`, `gitHubOrg`, `gitHubRepo`, `remoteName`, `upstreamURLBase`, `targetVersion`, `prefixes[].name/prefix/repo/summarise/consistentImages`) match the `Options`/`Prefix` structs in `cmd/generic-autobumper/main.go:121-170` and satisfy the other `validateOptions` checks (fork mode requires `gitHubToken`+`remoteName`, both present; `consistentImages: false` with no exceptions passes cross-field validation).
- `prefix: "gcr.io/k8s-staging-test-infra"` matches exactly the images in `.ko.yaml`'s `baseImageOverrides` (`alpine`, `git`, `git-custom-k8s-auth`); unrelated images (`gcr.io/distroless/static`, `google/cloud-sdk`) correctly untouched.
- `targetVersion: "latest"` is a supported value per `cmd/generic-autobumper/main_test.go`.
- All CI checks pass (image-build-test, integration, unit-test, race-detector, verify-lint, EasyCLA, Netlify preview) — none of them exercise `hack/autobump.yaml` itself, so they don't catch the missing-field bug.
- Low blast radius: new file isn't referenced anywhere else, so the bug can't regress existing behavior — it only means the bumper fails/no-ops wherever it's eventually invoked.

## Open questions
- None. All prior questions are answered under "Resolved": `includedConfigPaths` is no longer required when `extraFiles` is set, the scheduled job exists and mounts the expected token, and bot PR #1036 demonstrates the bump reaching a real PR.

## Followups

### tests: probe the `extraFiles`-only bump path used by `hack/autobump.yaml`
- where: `cmd/generic-autobumper/main_test.go`, `cmd/generic-autobumper/bumper/`, `hack/autobump.yaml`
- necessity: should
- mode: probe
- why followup: the config merged invalid and stayed inert until `7b32f3d6f` fixed `validateOptions`, yet no test exercises the exact shape it ships with (empty `includedConfigPaths`, `extraFiles: [".ko.yaml"]`). A throwaway probe is the cheapest way to look for remaining bugs in that path before investing in a permanent regression test.

```text
In `kubernetes-sigs/prow`, following merged PR #796 ("autobump prow base images", merge commit 85e45f6d25aa66f1ee071705b4f72a425fbaeed4), run a probe on the `cmd/generic-autobumper` code paths exercised by `hack/autobump.yaml` — the config shape with `includedConfigPaths` empty, `extraFiles: [".ko.yaml"]`, `targetVersion: latest`, and one prefix `gcr.io/k8s-staging-test-infra`.

Task: write throwaway Go tests (package `main`, reusing existing fakes such as `fakeImageBumperCli` and the temp-dir helpers in `cmd/generic-autobumper/main_test.go`) that exercise:
1. Loading the real `hack/autobump.yaml` into both `options` and `bumper.Options` (as `parseOptions` does) and calling `validateOptions`.
2. `updateReferences` with empty `IncludedConfigPaths` and `ExtraFiles: [".ko.yaml"]` against a temp `.ko.yaml`-shaped file whose `baseImageOverrides` contain `gcr.io/k8s-staging-test-infra/alpine:v20250101-aaaaaaaaaa`, `.../git:v20250101-aaaaaaaaaa`, `.../git-custom-k8s-auth:v20250101-aaaaaaaaaa`, and an unrelated `gcr.io/distroless/static:nonroot`. Assert all three staging tags are replaced with the fake latest tag and the unrelated image is untouched.
3. PR title/body generation (`PRTitleBody`/`generatePRBody`) when the only change is in an extra file, with `summarise: false`.
4. Dry-run reporting (`HasChanges`/`processDryRun`): a tree with only `.ko.yaml` modified must report a change and print a PR body; an unchanged tree must report no change.

Run `go test ./cmd/generic-autobumper/...`.

Acceptance criteria: every listed behaviour is exercised; each failure is triaged as a real implementation bug or a wrong expectation, with evidence (test name, actual vs expected). If a real bug is found, apply only the minimal fix in non-test code. If no real bug turns up, change no files. Report each failing case (input, actual, expected) so a regression test can be added later.

Scope guard: the probe tests are throwaway — delete them before finishing; the deliverable is bug fixes, never the tests. Do not fix test-only problems, style, or unrelated code. Do not modify `hack/autobump.yaml`, the validation logic, or anything under `kubernetes/test-infra`. Do not commit or push.
```
