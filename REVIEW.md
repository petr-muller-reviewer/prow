---
pr: kubernetes-sigs/prow#952
title: "fix(site): make the docs link checker check links"
head_sha: eff2b2dd20c14a7a31c0e477074e992ed8fa337d
base: main
reviewed_at: 2026-09-22T13:18:51Z
verdict: request-changes
refresh_log:
  - old_sha: eff2b2dd20c14a7a31c0e477074e992ed8fa337d
    new_sha: eff2b2dd20c14a7a31c0e477074e992ed8fa337d
    at: 2026-09-22T13:18:51Z
    summary: "No code changes; recorded Prucek's approval and the approval-notifier update."
---

## What this PR does
- Builds the documentation site, then scans the generated tree once with htmltest.
- Resolves absolute links from `public` and removes broad internal-link ignores.
- Pins htmltest in `hack/tools` and exposes a top-level `verify-links` target.
- Adds an internal-only htmltest target and excludes generated site output from unrelated checks.

Since previous review:
- No code changes: `eff2b2dd20c14a7a31c0e477074e992ed8fa337d` remains the PR head.
- Prucek submitted an `APPROVED` review with `/ok-to-test` at 2026-09-22T11:26:52Z; kubernetes-prow[bot] subsequently updated the approval status at 2026-09-22T11:26:59Z.

## Findings

### [blocking] Routine verifier runs the external-link scan
- where: `hack/make-rules/verify/links.sh:51`; `site/Makefile:33-38`
- concern: `verify-links` invokes `check-broken-links`, which runs htmltest without `-s`, rather than the new `check-broken-links-internal` target. `IgnoreExternalBrokenLinks` makes broken external links warnings but does not prevent external requests, so the normal verifier is still slow and network-dependent. Its final success message says that internal links were checked even though it ran a broader scan.
- excerpt: |
    # hack/make-rules/verify/links.sh
    make check-broken-links HTMLTEST="${HTMLTEST}"

    # site/Makefile
    check-broken-links-internal:
        $(HTMLTEST) -s -c .htmltest.yml

### [question] What will exercise this target before the planned CI job exists?
- where: `Makefile:109-111`; `hack/make-rules/verify/all.sh:26-70`
- concern: `verify-links` is a standalone target and is not called by `make verify` or an existing workflow. The PR deliberately defers CI because the current verification image lacks Hugo and Node; confirm that manual use is intended until a dedicated internal-link presubmit or periodic job lands.
- excerpt: |
    verify-links:
        hack/make-rules/verify/links.sh

### [question] Should the local Hugo requirement match the production build?
- where: `hack/make-rules/verify/links.sh:23-27`; `netlify.toml:7-9`
- concern: The verifier accepts any `hugo` on PATH, while Netlify uses Hugo 0.147.8. Validating the extended edition and version, or using a pinned environment, would make local results match the deployed build more closely.
- excerpt: |
    if ! command -v hugo >/dev/null 2>&1; then
      echo "ERROR: hugo is required to build the site before checking links."
    fi

## Checked
- The complete generated site is scanned once with `DirectoryPath: public`; the obsolete per-file wrapper is removed.
- `htmltest` is pinned in `hack/tools`; `go build -mod=readonly github.com/wjdp/htmltest` succeeds.
- The internal-only site target passes `-s` to htmltest as intended.
- `git diff --check` is clean.
- Generated `site/public` and `site/resources` output is excluded from the existing spelling and shell-permission checks.

## Open questions
- Is `make verify-links` intended to be the deterministic internal-only check, with a separately named manual or periodic target for external URLs?
- Until a CI image with Hugo and Node is selected, who or what is expected to run `verify-links`?
- Should the verifier enforce the Hugo extended version used by Netlify?
