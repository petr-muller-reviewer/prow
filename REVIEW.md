---
pr: kubernetes-sigs/prow#796
title: "autobump prow base images"
head_sha: ab99df655a530e6e7b30a29efad116423e4d8018
base: main
reviewed_at: 2026-07-26T23:35:11Z
verdict: request-changes
---

## What this PR does

- Adds `hack/autobump.yaml`, a new config for the in-tree `cmd/generic-autobumper` tool.
- Targets images under the `gcr.io/k8s-staging-test-infra` prefix, referenced as base images in `.ko.yaml`, via `extraFiles: [".ko.yaml"]`.
- Fixes #559: those images now expire after 6 months and `.ko.yaml` hadn't been bumped since ~July 2025.
- Purely additive: one new file, not referenced by any existing build/test/deploy path.

## Findings

### [blocking] missing `includedConfigPaths` causes the bumper to fail on every invocation
- where: `hack/autobump.yaml` (whole file); validated against `cmd/generic-autobumper/main.go:123,232-234`
- concern: `IncludedConfigPaths []string \`yaml:"includedConfigPaths"\`` is unconditionally required by `validateOptions`, which returns `errors.New("includedConfigPaths is mandatory")` when empty. `main()` calls `validateOptions` before any bump logic runs and `logrus.Fatalf`s on error. This config never sets `includedConfigPaths`, so any invocation (`generic-autobumper --config=hack/autobump.yaml`) exits nonzero immediately, before `.ko.yaml` (reached only via `extraFiles`, processed independently) is ever touched. As written this cannot fix #559 — the bumper is dead on arrival. Confirmed directly by reading `main.go:232-233`; not inferred.
- excerpt: |
    // cmd/generic-autobumper/main.go
    if len(o.IncludedConfigPaths) == 0 {
        return errors.New("includedConfigPaths is mandatory")
    }

    # hack/autobump.yaml — missing field
    extraFiles:
      - ".ko.yaml"
    prefixes:
      - name: "k8s-staging-test-infra GCR images"

### [question] no periodic job wired to run this config
- where: `hack/autobump.yaml` (whole file)
- concern: Nothing in this repo (`config/jobs/**`, `Makefile`s, `.github/workflows/**`) invokes `generic-autobumper --config=hack/autobump.yaml` on a schedule. The periodic + postsubmit jobs likely need to live in a separate infra/job-config repo, but that's invisible from here. Even once the blocking issue is fixed, it's unclear the bump will actually run without that wiring, and #559 could get marked resolved while the automation stays inert.
- excerpt: |
    gitHubToken: "/etc/github-token/token"
    ...
    prefixes:
      - prefix: "gcr.io/k8s-staging-test-infra"

## Checked
- All other fields (`gitHubLogin`, `gitHubToken`, `gitName`, `gitEmail`, `skipPullRequest`, `gitHubOrg`, `gitHubRepo`, `remoteName`, `upstreamURLBase`, `targetVersion`, `prefixes[].name/prefix/repo/summarise/consistentImages`) match the `Options`/`Prefix` structs in `cmd/generic-autobumper/main.go:121-170` and satisfy the other `validateOptions` checks (fork mode requires `gitHubToken`+`remoteName`, both present; `consistentImages: false` with no exceptions passes cross-field validation).
- `prefix: "gcr.io/k8s-staging-test-infra"` matches exactly the images in `.ko.yaml`'s `baseImageOverrides` (`alpine`, `git`, `git-custom-k8s-auth`); unrelated images (`gcr.io/distroless/static`, `google/cloud-sdk`) correctly untouched.
- `targetVersion: "latest"` is a supported value per `cmd/generic-autobumper/main_test.go`.
- All CI checks pass (image-build-test, integration, unit-test, race-detector, verify-lint, EasyCLA, Netlify preview) — none of them exercise `hack/autobump.yaml` itself, so they don't catch the missing-field bug.
- Low blast radius: new file isn't referenced anywhere else, so the bug can't regress existing behavior — it only means the bumper fails/no-ops wherever it's eventually invoked.

## Open questions
- Add `includedConfigPaths` (e.g. `["."]` or the actual directories to scan) — without it the tool exits nonzero on every run.
- Has this config been dry-run (`generic-autobumper --config hack/autobump.yaml`) to confirm it validates and reaches the bump step?
- Is there a companion PR/job (likely in a separate infra repo) that actually invokes this config on a schedule, and does it mount the `/etc/github-token/token` secret this config expects?
