---
issue: kubernetes-sigs/prow#815
title: "Prow issue: this-is-fine is documented but not working"
state: open
labels: kind/bug, area/plugins
main_sha: 850dcba9be8689410a63a24ee741664ecf965494
triaged_at: 2026-08-20T13:02:00Z
verdict: accepted
refresh_log:
  - at: 2026-07-29T01:04:43Z
    summary: Two new comments (t-inu, nasa9084) confirming the doc reference and long-standing breakage; no state/label change, no linked PRs — updated in place.
  - at: 2026-08-03T22:16:34Z
    summary: Labels applied (kind/bug, area/plugins); krzyzacy (original bucket author) confirmed ownership and asked BenTheElder for a community-owned bucket alternative — confirms existing provenance finding, updated in place.
  - at: 2026-08-09T13:28:11Z
    summary: BenTheElder suggested asking upodroid/sig-k8s-infra about a community-owned bucket; no state/label change — updated in place.
---

## Findings

### [reproducibility] `/this-is-fine` and siblings are 100% reproducible, not timing-dependent
- detail: Commenting `/this-is-fine` on any issue/PR triggers no bot response. Verified independently of the report by tracing the code path and confirming the underlying image URLs return HTTP 403.
- evidence: `curl -sI https://storage.googleapis.com/this-is-fine-images/this_is_fine.png` → `HTTP/2 403`. Same result for `this_is_not_fine.png` and `this_is_unbearable.jpg`.

### [cause] Hardcoded GCS bucket `this-is-fine-images` is dead (403 on all objects), and was never community-owned infra
- detail: `pkg/plugins/dog/dog.go:46` hardcodes `defaultFineImagesRoot = "https://storage.googleapis.com/this-is-fine-images/"` as an unconfigurable Go `const` (not plugin config). The bucket now returns 403/401 on every object and on bucket-level metadata queries; `gsutil`/GCS JSON API responses are the generic "AccessDenied ... or it may not exist" (GCS does not distinguish "deleted" from "exists but private" for anonymous callers, so this can't be resolved further without GCP project-owner access).
- evidence: `pkg/plugins/dog/dog.go:46`; `curl https://storage.googleapis.com/storage/v1/b/this-is-fine-images` → 401 `AccessDenied`; `curl https://storage.googleapis.com/this-is-fine-images/` → `AccessDenied`.
- history: traced via `kubernetes/test-infra` history (this code predates the `kubernetes-sigs/prow` split, carried over verbatim in commit `db89760fe`, "move pkg, cmd & test folders in inside prow to the root of the repo (#101)"). The feature was added by Sen Lu (`krzyzacy`) via [kubernetes/test-infra#9833](https://github.com/kubernetes/test-infra/pull/9833) ("support /this-is-fine"), in two same-day commits: `c18f4131` (2018-10-17, "this-is-fine") initially served images from `https://dogechatbot.appspot.com/static/...` — a personal App Engine app (1:1 with an individual's own GCP project) — then `aa11eb48` (same day, "switch image hosting to gcs") moved to the `this-is-fine-images` bucket, with no discussion in the PR of which GCP project backs it. A GitHub code search of `kubernetes/k8s.io` (the repo that registers all community-owned GCP projects/buckets) for `this-is-fine-images` returns zero hits, meaning this bucket was never onboarded as shared SIG-Testing/k8s-infra infrastructure. `krzyzacy` has had no visible upstream Kubernetes activity in years. Taken together, this strongly suggests the bucket was an individual contributor's ad hoc personal-project asset that lapsed (e.g. GCP project cleanup/expiration) rather than a piece of infra with a current owner to ask for renewal. Wayback Machine has a single crawl of the image URL, 2022-05-17, returning HTTP 200 — the sole data point bounding when it broke (sometime between 2022-05-17 and the 2026-07-29 report); no crawls exist to narrow this further.
- evidence: `git log --all --oneline -S "this-is-fine-images"` (in a `kubernetes/test-infra` checkout) → `aa11eb48af47ef87f6ac3fe9ee29332dbaa4bab5`; PR https://github.com/kubernetes/test-infra/pull/9833; `gh api "search/code?q=this-is-fine-images+repo:kubernetes/k8s.io"` → no results; `curl "http://web.archive.org/cdx/search/cdx?url=storage.googleapis.com/this-is-fine-images*&output=json"` → single row, `20220517001514`, status `200`.

### [cause] Failure path is silently swallowed — no error surfaced to the user
- detail: `readDog()` skips the random-dog API fetch when a URL is already supplied and calls `github.ImageTooBig()`, which does an HTTP HEAD and treats any non-200 response as an error (mislabeled "too big", but the effect is the same). `handle()` retries 5 times against the same dead URL, then returns a generic `errors.New("could not find a valid dog image")` that is logged server-side only — no comment is ever posted back to the issue/PR, so the command appears to silently do nothing.
- evidence: `pkg/github/helpers.go:54-70` (`ImageTooBig`); `pkg/plugins/dog/dog.go:138-175` (`handle`), specifically the retry loop at `dog.go:165-172`.

### [related-code] All three "fine" commands share the broken root, not just the one in the issue title
- where: `pkg/plugins/dog/dog.go:36-51`
- excerpt: |
    fineRegex       = regexp.MustCompile(`(?mi)^/this-is-fine\s*$`)
    notFineRegex    = regexp.MustCompile(`(?mi)^/this-is-not-fine\s*$`)
    unbearableRegex = regexp.MustCompile(`(?mi)^/this-is-unbearable\s*$`)
    ...
    defaultFineImagesRoot = "https://storage.googleapis.com/this-is-fine-images/"

### [related-code] `/woof` and `/bark` are unaffected — different code path
- where: `pkg/plugins/dog/dog.go:100-132` (`readDog`), `dog.go:144-159` (`handle` dispatch)
- excerpt: |
    if dogURL == "" {
        // fetches from https://random.dog/woof.json — live, unrelated service
        ...
    }
- relevance not applicable (code finding, not issue/PR ref); explains why the reporter observed partial functionality ("other commands like meow is working well").

### [related-code] Existing unit tests likely mock the HTTP layer, masking the live breakage
- where: `pkg/plugins/dog/dog_test.go`
- excerpt: |
    (tests exercise `pack`/`githubClient` interfaces with fakes, not real HTTP calls to the GCS bucket)

### [related-code] Comments are posted as plain markdown image links, not native GitHub attachments
- detail: `FormatURL()` composes the comment body as `[![dog image](%s)](%s)` — a markdown image link GitHub renders inline by fetching the URL client-side. There is no REST/GraphQL endpoint for a bot to upload a file and have GitHub host it as a native attachment (the web UI's drag-and-drop attachment flow uses an internal, session-authenticated endpoint, not available to PAT/App tokens). This constrains the fix: whatever hosts the replacement images, the plugin still needs to hand GitHub a public URL — a binary-embedded image alone doesn't produce one.
- where: `pkg/plugins/dog/dog.go:95-98` (`FormatURL`)

Since previous triage (2026-07-29T01:04:43Z):
- `t-inu` pointed at the public docs entry (`https://prow.k8s.io/command-help#woof`) confirming the command is documented as `/(woof|bark|this-is-{fine|not-fine|unbearable})` — matches the existing finding, no new information.
- `nasa9084` reported that the command "was working two years ago" but now doesn't even show an image already posted, linking to a 2024-08-26 `/this-is-fine` comment on `kubernetes-sigs/contributor-playground#1375`. That comment's embedded image now 403s too — consistent with (and further confirming) the bucket-wide 403 finding rather than contradicting it: the breakage affects historical posts, not just new invocations.

Since previous triage (2026-08-03T22:16:34Z):
- Labels `kind/bug` and `area/plugins` were applied to the issue (no `good-first-issue`).
- `krzyzacy` — the bucket's original author per the traced `kubernetes/test-infra#9833` history — commented directly on the issue ("crap lol...") and asked `@BenTheElder` "any community owned bucket we can borrow :-)". This confirms the provenance finding (the bucket was indeed their own asset, now lapsed) directly from the source, and surfaces a maintainer-driven alternative to vendoring: hosting the images in an existing community-owned k8s-infra bucket instead. No bucket has been offered yet as of this refresh.

Since previous triage (2026-08-09T13:28:11Z):
- `BenTheElder` replied, suggesting `@upodroid` / sig-k8s-infra as a contact for a community-owned bucket. Still no bucket offered or confirmed — the maintainer-driven alternative remains open, not resolved.

## Checked

- Confirmed all three "fine" images return 403 (`this_is_fine.png`, `this_is_not_fine.png`, `this_is_unbearable.jpg`), not just the one named in the issue title.
- Confirmed `/woof` and `/bark` use an unrelated live API (random.dog) and are unaffected — ruled out a shared/systemic plugin-dispatch bug.
- Ruled out plugin-configuration issue (e.g. `dog` plugin disabled for the reporter's repo) — the reporter's thread shows other bot interactions working, and the command is silently ignored rather than rejected as unrecognized, consistent with the traced code path rather than a config gap.
- Checked git history of `pkg/plugins/dog/dog.go`: no recent change to the URL/constants in this repo; this is an external (bucket-side) failure, not a code regression in `kubernetes-sigs/prow`.
- Traced the bucket's provenance through `kubernetes/test-infra` history (PR #9833, commits `c18f4131`/`aa11eb48`) and confirmed via `kubernetes/k8s.io` code search that it was never registered as community-owned infra — no current SIG-Testing/k8s-infra owner exists to ask for renewal.
- Confirmed via `dog.go:95-98` (`FormatURL`) that comments are plain markdown image links, not native attachments — ruling out a "post as attachment instead of URL" fix; GitHub's bot API has no such capability.
- Checked Wayback Machine for the bucket URL: only one crawl on record (2022-05-17, HTTP 200), insufficient to pin down exactly when it broke.

## Next steps

- Do not pursue restoring/re-requesting access to the GCS bucket — it was an individual contributor's personal-project asset, never SIG-Testing/k8s-infra-owned, so there is no current owner to ask.
- Recommended fix: vendor the three images into the repo (e.g. `pkg/plugins/dog/images/`) and point `defaultFineImagesRoot` (`dog.go:46`) at their `raw.githubusercontent.com` URL — no new external dependency, no infra provisioning needed, and matches an existing in-repo pattern of fetching content via `raw.githubusercontent.com` (`pkg/plugins/testfreeze/checker/checker.go:42`). Update `dog_test.go` to pin the new source.
- Alternative considered and available if a fully self-contained solution is preferred: `go:embed` the images into the `hook` binary and add a small HTTP handler serving them at a stable Prow-owned URL (e.g. under `prow.k8s.io`), removing even the `raw.githubusercontent.com` dependency. More invasive (new route in a shared service) — only worth it if the team wants zero external dependencies rather than accepting GitHub's own CDN.
- Not viable: posting images as native GitHub comment attachments — no bot-accessible API for this (see `related-code` finding above); rule this out if it comes up during discussion.
- Labels `kind/bug` and `area/plugins` have been applied (2026-08-03). Consider also adding `good-first-issue` — the vendoring fix remains fully actionable without waiting on any maintainer infra decision, regardless of whether a community bucket materializes.
- Optional follow-up (separate issue): `handle()` in `pkg/plugins/dog/dog.go` swallows failures into a log-only error with no user-facing feedback; consider surfacing a comment when a matched command fails outright, across the plugin generally.

## Open questions

- If vendoring via `raw.githubusercontent.com`: pin the URL to a specific commit SHA vs. `main`? `main` auto-follows file moves/renames, but a future repo restructure could silently break the link the same way the GCS bucket did; a pinned SHA is more failure-proof but needs manual updating if the plugin package ever moves.
- Is a fully self-contained (`go:embed` + Prow-served URL) approach worth the extra implementation cost versus depending on GitHub's own raw-content CDN?
- If BenTheElder (or another maintainer) offers a community-owned bucket in response to `krzyzacy`'s ask: does that change the recommendation away from vendoring/`go:embed`, or is vendoring still preferred as the simpler, dependency-free fix regardless? Worth resolving before a PR is opened, to avoid rework. As of 2026-08-20, this is pointed at sig-k8s-infra (`upodroid`) but not yet resolved — no bucket has materialized.
