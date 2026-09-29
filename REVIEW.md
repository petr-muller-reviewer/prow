---
pr: 708
title: "external-plugins: add netlify-preview plugin to retry deploy previews"
head_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
base: main
reviewed_at: "2026-07-28T23:05:37Z"
verdict: APPROVE_WITH_SUGGESTIONS
gate:
  decision: merge
  gated_at: "2026-06-30T12:03:32Z"
  gated_head_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
  reviewed_head_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
refresh_log:
  - old_sha: 11b326397a5be0d369e95f680b1522f1375afd01
    new_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
    refreshed_at: "2026-06-02T08:58:17Z"
    summary: "Force-push (same content); author self-review comments on config.go; /retest triggered"
  - old_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
    new_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
    refreshed_at: "2026-06-30T13:00:47Z"
    summary: "No code changes; petr-muller submitted COMMENTED review with 8 inline comments; Caesarsage cc'd stmcginnis"
  - old_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
    new_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
    refreshed_at: "2026-07-28T23:05:37Z"
    summary: "No code changes; second consumer (Kueue) surfaced via apullo777/mimowo; Caesarsage committed to quiet /retest, rename to /netlify-rebuild, and pagination as follow-up work"
recommended_rereview:
  - at: "2026-08-17T22:11:26Z"
    old_sha: 35fd83923ae5a1ad2d8f65a7bb411ae1d9e695d7
    new_sha: e8849971aad14f136ab4b965d06a9fe679b9115b
    reason: "390 net lines rewriting /retest-quieting, ListDeploys pagination, and command rename — exactly the areas of 3 converging findings"
---

# PR #708 Review: external-plugins: add netlify-preview plugin

**Author**: Caesarsage · **Branch**: `netlify-command` → `main` · +1252 −0 across 11 files · 6 commits

## Re-review Recommended

### 2026-08-17T22:11:26Z — `35fd83923` → `e8849971a`

**What changed** (single commit, "address review feedback", 8 files, +447/−57, 390 net lines):

- `config/config.go` (+8/−8) — renamed `Repo` type to `SiteConfig` per singh1203's naming nit.
- `plugin/plugin.go` (+10/−10) — renamed `/rebuild-preview` command and `RebuildPreviewCommand` constant to `/netlify-rebuild` / `NetlifyRebuildCommand`, per petr-muller's and the Kueue thread's naming suggestion.
- `netlify/client.go` (+69/−12, net +57) — **replaced `ListDeploys` entirely** with a new `ForEachDeployPage` method: paginates via the Netlify `Link` response header (new `nextLink` header-parser), narrows server-side with `deploy-previews=true`, caps at `maxListDeployPages = 10` pages of `per_page=100`, and early-exits the page walk once a match is found.
- `server.go` (+44/−28, net +16) — **rewrote the comment-response logic**: introduced a `noopLevel` (Debug for `/retest`, Info for `/netlify-rebuild`) so log verbosity differs by command, and gated every `s.comment(...)` call behind `command == NetlifyRebuildCommand` — `/retest` no longer posts any PR comment on the untrusted-user, missing-config, already-retried, or no-op paths; only `/netlify-rebuild` does. `netlifyClient` interface signature changed (`ListDeploys` → `ForEachDeployPage`).
- `netlify/client_test.go` (+149/−~few), `server_test.go` (+178/−~few), `plugin_test.go` (+31/−17) — substantial new test coverage for pagination, the new command name, and per-command comment gating.
- `site/content/.../netlify-preview.md` (+15/−~few) — docs updated for the renamed command.

**Why this exceeds "update in place":** This is not a targeted fix — it's a rewrite of the exact three converging concerns the review flagged (`ListDeploys` pagination, `/retest` overlap noise, and the command name), each with a genuinely new mechanism: a hand-rolled `Link`-header parser and page-capped iteration protocol that didn't exist before, and a behavior change where `/retest` goes from "always comments" to "never comments, logs at Debug instead." Both mechanisms carry their own new edge cases (off-by-one in `maxListDeployPages`, malformed `Link` headers, whether early-exit-on-first-match correctly handles a PR with multiple deploys across pages, whether the trusted-user deny path silently swallowing `/retest` is desired) that the prior review never evaluated because the code didn't exist yet. The `netlifyClient` interface's public shape also changed. This satisfies the ">100 lines net change AND touches areas of existing findings" trigger.

**Activity since the last review** (`2026-07-28T23:05:37Z`):
- **2026-08-13** — Caesarsage pushed the above commit and left an inline comment on `server.go` explaining the pagination design: paginated via `Link` header, 100/page max, no server-side `review_id` filter so "some walking is unavoidable," early-exit on first match, 10-page cap, `deploy-previews=true` narrows server-side.
- **2026-08-13** — Caesarsage submitted a COMMENTED review (no body) alongside that comment.
- **2026-08-13** — kubernetes-prow[bot] posted the routine "NOT APPROVED" approval-notifier comment (no reviewer/approver activity to act on).

**Preliminary findings against `e8849971a`** (from `/code-review`, medium effort — supersedes the need for a from-scratch look at these specific spots when the full re-review happens):

- **`netlify/client.go:116`** — `nextLink` splits the raw `Link` header on literal commas instead of using a proper link-header parser (cf. `pkg/github/links.go`). If a `Link` header ever contains a comma inside a URL, the split breaks that URL, the bracket check fails, `nextLink` returns `""`, and `ForEachDeployPage` silently stops paginating early — a real deploy on a later page gets missed with no error. New code from this commit; not covered by any prior finding.
- **`plugin/plugin.go:37`** — `commandRe` matches `/retest` case-insensitively (`(?mi)`), diverging from `pkg/pjutil.RetestRe`'s case-sensitive `(?m)^/retest\s*$` used by the trigger plugin. A commenter writing `/ReTest` retries the Netlify preview but not CI — inconsistent behavior for what looks like the same command. Pre-existing regex, but its interaction with the now-more-consequential `/retest` no-op path deserves a look during re-review.
- **`server.go` (5 call sites: ~118, 135, 144, 184, 191)** — the new "only `/netlify-rebuild` gets a comment reply" rule from this commit is repeated as `if command == netlifypreview.NetlifyRebuildCommand { return s.comment(...) }; return nil` five separate times in `handleIssueComment`, instead of being expressed once. A future new early-return path can easily forget the gate, silently making `/retest` chatty again or `/netlify-rebuild` silent. New pattern introduced by this commit — a maintainability concern on the exact code this refresh flagged for re-review.
- **`plugin/plugin.go:39`** — `ParseCommand` uses `FindStringSubmatch` (first match only), so a comment body with both `/retest` and `/netlify-rebuild` on separate lines only honors the first one found; the second is silently dropped. Pre-existing behavior, now more consequential since the two commands diverge sharply in comment/logging behavior post-rewrite.

Two additional `/code-review` findings restate ground already tracked in this file and aren't repeated here: config key format not validated (`config/config.go:49`, cf. "Config key format not validated" above) and config not hot-reloaded (`main.go:119`, cf. "Config loaded once at startup" above); a fifth finding (duplicate `flagutil` import, `main.go:34`) restates "Redundant flagutil import" above at an updated line number.

## Gate

**MERGE** — gated 2026-06-30 at `35fd83923`

Purely additive external plugin with zero blast radius on existing Prow installations. All 12 review findings remain unaddressed (no code changes since the review), but none are blockers — they are quality improvements to a brand-new plugin that only activates when explicitly deployed. No GitHub reviewer has submitted `CHANGES_REQUESTED`. The pagination gap in `ListDeploys` is the most impactful item and should be a fast follow-up, but it affects only new adopters of this plugin, not existing deployments.

### Findings disposition (Area 1)

All findings from the original review stand — SHA unchanged since review. None gate merge:

- **ListDeploys pagination (3/3 converge)** — not addressed. Functional limitation of the new plugin; causes false "no preview found" on busy repos. Recommend immediate follow-up, but only affects new adopters.
- **/retest overlap (3/3 converge)** — not addressed. Deliberate design choice per #693. Noisier than ideal but not a correctness issue.
- **Silent API failures (2/3 converge)** — not addressed. UX gap for new adopters; not a regression.
- **Redundant flagutil import (2/3 converge)** — not addressed. Code hygiene only.
- **4 moderate findings** (dead validation, wrong logger/level, test name, webhook error) — not addressed. All code quality items with no external impact.
- **4 minor findings** — not addressed. Suggestions for future improvement.
- **singh1203 inline comment** (Config type naming) — discussed, author explained rationale. Not blocking.

### Merge risk (Area 2)

No notable merge risk.

- **API changes**: None. All code is new under `cmd/external-plugins/netlify-preview/`. No existing exported surfaces modified.
- **Configuration changes**: None. The only change to an existing file is `.prow-images.yaml` — one line added to register the image build alongside existing external plugins. No effect on other images or builds.
- **Behavioral changes**: None. The plugin is inert until an operator explicitly deploys it and registers it in their `external_plugins` config. Dry-run defaults to `true`.

### Gating list

Nothing gates merge. The strongest candidate — `ListDeploys` pagination — is a functional limitation of a new, opt-in plugin, not a backward-incompatible change or regression.

---

## Maintainer Advisor — Final Recommendation

**APPROVE WITH SUGGESTIONS**

Well-structured, purely additive external plugin with low deployment risk and no modifications to existing Prow components. All three reviewers converged on the same set of minor concerns, none of which represent correctness bugs or security issues. The code follows established Prow external plugin patterns closely, has good test coverage, and the remaining issues are improvements that can be addressed in follow-up work without blocking the merge.

### Suggested PR Comment

> This is a clean, well-structured external plugin that follows established Prow patterns. All three review dimensions (code quality, maintainability, deployment risk) came back positive. The main item I would like to see addressed — either before merge or as an immediate follow-up — is adding `per_page=100` to the `ListDeploys` call, since the current default of 20 can cause false "no preview found" results on active repos. The `/retest` overlap with trigger is fine as a design choice but deserves a code comment explaining the intent. Approving with suggestions; none of the remaining items are blockers.

## Reviewer Verdicts

| Code Quality | Maintainability | Deployment Risk |
|---|---|---|
| APPROVE | COMMENT | LOW RISK |
| Senior Go Engineer | Burden: LOW | Purely additive |

## Stats

- **4** Converging (2+/3 reviewers)
- **4** Moderate
- **6** Minor
- **8** Positive

## Files Changed

- `.prow-images.yaml`
- `cmd/external-plugins/netlify-preview/config/config.go`
- `cmd/external-plugins/netlify-preview/config/config_test.go`
- `cmd/external-plugins/netlify-preview/main.go`
- `cmd/external-plugins/netlify-preview/netlify/client.go`
- `cmd/external-plugins/netlify-preview/netlify/client_test.go`
- `cmd/external-plugins/netlify-preview/plugin/plugin.go`
- `cmd/external-plugins/netlify-preview/plugin/plugin_test.go`
- `cmd/external-plugins/netlify-preview/server.go`
- `cmd/external-plugins/netlify-preview/server_test.go`
- `site/content/en/docs/components/external-plugins/netlify-preview.md`

---

## Converging Concerns (2+ reviewers)

### [3/3] ListDeploys has no pagination — may miss deploys on busy repos

**Location**: `netlify/client.go:65`

The Netlify `GET /api/v1/sites/{site_id}/deploys` endpoint returns paginated results (default 20 per page). For active repos like `kubernetes/website`, the latest deploy preview for a given PR may not appear in the first page if many production or branch deploys have occurred since.

Without a `per_page` query parameter or pagination loop, the plugin may falsely report "No Netlify deploy preview was found" for PRs that do have one.

**Highest-confidence issue across all three reviews.** The maintainer advisor recommends this be addressed either before merge or as an immediate follow-up.

> **Suggestion**: At minimum, pass `?per_page=100` as a query parameter to widen the window. Ideally, also filter by `branch=deploy-preview-{number}` if the Netlify API supports it. Consider paginating if the first page yields no match.

---

### [3/3] /retest overlaps with built-in trigger plugin

**Location**: `plugin/plugin.go:37,69` · `server.go:131`

Prow's built-in `trigger` plugin already handles `/retest` to re-run failed presubmit jobs (`pkg/plugins/trigger/trigger.go:140`). On repos where this external plugin is enabled, **both plugins fire** on every `/retest` comment. This is a deliberate design choice (per #693), but creates implicit coupling: any future change to `/retest` semantics in trigger must consider this plugin and vice versa.

Consequences:
- Every `/retest` hits the Netlify API to list deploys, even when the user only wants to re-run Prow CI.
- The plugin posts a comment on every `/retest` where Netlify has nothing useful to do (preview is ready, no preview exists, etc.) — the *common case* — which is noisy.
- On `kubernetes/website` with frequent `/retest` usage, this means an extra bot comment on most retests.

> **Suggestion**: Either (a) only post a comment when a retry is actually performed — stay silent on no-ops for `/retest` specifically, or (b) use a distinct command like `/retest-preview` instead of overloading `/retest`. At minimum, add a code comment on `commandRe` explaining the intentional overlap with the trigger plugin, and document the dual-processing behavior.

---

### [2/3] Silent failure when Netlify API calls fail

**Location**: `server.go:165`

When `RetryDeploy` (or `ListDeploys`) fails, the error is returned and logged by the goroutine, but **no comment is posted to the PR**. The user gets complete silence — they don't know whether the retry was attempted, succeeded, or failed.

> **Suggestion**: Post a comment like "Failed to communicate with Netlify API" before returning the error.

---

### [2/3] Redundant flagutil import

**Location**: `main.go:189-190`

`flagutil` is imported twice — once bare and once aliased as `prowflagutil`. They reference the same package path.

```go
"sigs.k8s.io/prow/pkg/flagutil"
prowflagutil "sigs.k8s.io/prow/pkg/flagutil"
```

> **Suggestion**: Drop the bare import and use `prowflagutil.OptionGroup` in `Validate()`, or drop the alias and use `flagutil` everywhere.

---

## Moderate

### Dead validation for --netlify-api-url

**Location**: `main.go:224-226`

The `--netlify-api-url` flag has a default value of `https://api.netlify.com`, but `Validate()` rejects an empty value. Since the default is always set by `flag.StringVar`, the empty-string check can never trigger.

> **Suggestion**: Remove the validation check for `netlifyAPIURL`.

---

### Goroutine error logging at Info level + wrong logger

**Location**: `server.go:86-90`

Two issues in the goroutine error path:

- **Wrong logger**: Error is logged via `s.log` instead of the request-scoped `l`, losing the event-type and event-GUID structured fields.
- **Wrong level**: Operational failures logged at `Info` level should be `Warn` or `Error` for monitoring visibility.

```go
go func() {
    if err := s.handleIssueComment(l, ic); err != nil {
        s.log.WithError(err).WithFields(l.Data).Info("Failed to handle issue comment.")
    }
}()
```

> **Suggestion**: Use `l.WithError(err).Error("Failed to handle issue comment.")` to preserve structured fields and use an appropriate log level.

---

### Misleading test name

**Location**: `server_test.go:156`

`TestHandleIssueCommentFailsClosedWhenMappingMissing` — the name says "fails closed" but the test verifies behavior when the repo-to-site config mapping is absent.

> **Suggestion**: Rename to `TestHandleIssueCommentReportsMissingMapping`.

---

### Webhook validation error discarded

**Location**: `server.go:64`

The fifth return value (error) from `github.ValidateWebhook` is discarded with `_`. Logging it would help debug webhook authentication failures in production.

> **Suggestion**: `if err != nil { s.log.WithError(err).Debug("Webhook validation failed.") }`

---

## Minor

### Unbounded io.ReadAll on error responses

**Location**: `netlify/client.go:78, 99`

On non-2xx responses, `io.ReadAll(resp.Body)` reads the entire body with no size limit.

> **Suggestion**: Use `io.LimitReader(resp.Body, 4096)` to cap the read.

---

### Missing server-level test coverage for edge paths

**Location**: `server_test.go`

No server-level test for `responseForDecision` or the `ActionUnsupportedState` path. Also no test for the dry-run path where `RetryDeploy` is skipped but the comment is still posted.

---

### Canceled/skipped Netlify build states not distinguished

**Location**: `plugin/plugin.go` (`Evaluate`)

**Source**: apullo777 (Kueue), 2026-07-27, PR comment

Kueue uses a Netlify `build.ignore` rule to skip previews for certain PRs, which produces canceled/skipped deploy states. These currently fall into `Evaluate()`'s generic unsupported-state path rather than being handled explicitly, so a user retrying a deliberately-skipped preview gets the same response as any other unsupported state, with no indication the skip was intentional.

Not blocking for the current single-consumer (kubernetes/website) scope, but relevant if this plugin is adopted more broadly.

---

### Config key format not validated

**Location**: `config/config.go:51-56`

`Validate()` checks that `site_id` is non-empty but does not verify that config keys match `org/repo` format. A typo like `website` instead of `kubernetes/website` would silently produce a config that never matches.

> **Suggestion**: Validate that each key contains exactly one `/` separator.

---

### Config loaded once at startup (no hot-reload)

**Location**: `main.go` · `config/config.go`

The plugin config is loaded at startup via `os.ReadFile`, not dynamically reloaded. Adding new repos requires a pod restart. Not blocking for initial adoption.

---

### Single Netlify token assumed for all configured repos

**Location**: `config/config.go` · `main.go` (`--netlify-token-file`)

**Source**: apullo777 (Kueue), 2026-07-27, PR comment

The plugin takes one Netlify API token for the whole process; the per-repo config only maps `site_id`. This is fine for the current single-consumer scope (kubernetes/website) but would need to become per-repository credentials if a second consumer (e.g. Kueue, using a separate Netlify project) can't share the same token. Caesarsage has explicitly deferred this until credential sharing is confirmed one way or the other.

---

## Positive Observations

- **Clean separation of concerns**: config, Netlify client, command parsing/decision logic, and server handler in distinct packages with clear interfaces.
- **Interfaces at consumer**: `githubClient`, `netlifyClient`, `pluginConfigAgent` defined at the consumer site (`server.go`), following the Go idiom of "accept interfaces, return structs." Narrow and specific.
- **Pure decision logic**: `Evaluate` is a pure function with no side effects — trivially testable. Decision/action pattern cleanly separates "what to do" from "doing it."
- **Trust gating reuses existing Prow infrastructure**: `trigger.TrustedUser` and `trigger.TrustedPullRequest` correctly reused rather than reimplemented.
- **Code block stripping**: `markdown.DropCodeBlock` correctly used before parsing commands, preventing accidental triggers from code examples.
- **Good test coverage**: All layers tested — config loading, Netlify client with mock HTTP transport, command parsing edge cases, decision logic for all state/command combinations, server-level integration tests. Behavior-oriented, not implementation-detail-oriented.
- **Follows existing plugin conventions**: Project structure, flag handling, health endpoint, and webhook validation all match `needs-rebase` and other external plugins.
- **Safe defaults**: Dry-run defaults to `true`, `url.PathEscape` for URL construction, `context.WithTimeout` for external API calls, strict YAML unmarshalling.

---

## Deployment Notes

- Existing Prow installations are **completely unaffected**. This plugin only activates when explicitly deployed and registered in `external_plugins`.
- If both trigger and this plugin are active on the same repo, `/retest` will fire both independently. Expected behavior but operators should be aware.
- Dry-run defaults to `true`, providing a safe initial deployment experience.
- New adopters must provision: Netlify API token, config YAML mapping repos to site IDs, HMAC webhook secret.

## Activity since 2026-05-30

- **2026-05-31** — Caesarsage submitted two self-review inline comments on `config/config.go`:
  - Proposed a doc comment for the `Config` type ("Config is the configuration wrapper to the repository...")
  - Note on the copyright year commit: "reverting this, adding Years is no longer supported"
- **2026-05-31** — Force-push: `11b326397` → `35fd83923` (same tree content, no code changes)
- **2026-06-01** — Caesarsage posted `/retest` to trigger CI on the updated head
- **2026-06-04** — Caesarsage cc'd @stmcginnis asking for a review
- **2026-06-30** — petr-muller submitted a COMMENTED review with 8 inline comments:
  - `main.go`: config hot-reload will eventually be needed (aligns with existing "Config loaded once at startup" finding)
  - `server.go`: deny comments on untrusted users should only fire for `/rebuild-preview`, not `/retest` (2 comments)
  - `server.go`: ListDeploys pagination concern — is the underlying API paginated? (aligns with existing "ListDeploys has no pagination" finding)
  - `server.go`: success comment may be unnecessary noise, GH status context may be enough
  - `server.go`: goroutine log level should be `Debug` for `/retest` events where no action is expected
  - `plugin/plugin.go`: targeted command should mention netlify, like `/netlify-rebuild` or `/netlify-preview`
  - `server.go`: `/retest` responses are too noisy — only respond to the targeted command
- **2026-07-27** — apullo777 (Kueue) opened a detailed cross-reference from kubernetes-sigs/kueue#13474, proposing Kueue as a second consumer of this plugin and reinforcing several open points:
  - `/retest` noise: Kueue sees frequent `/retest` for unrelated flakes while the preview is already green — wants the netlify-specific rebuild command kept separate and quiet
  - Pagination: relevant at Kueue's PR volume, since older PRs' previews may fall off the first page
  - Additional preview states: Kueue's `build.ignore` rule produces canceled/skipped Netlify builds, which currently fall into `Evaluate()`'s unsupported-state path
  - Credential configuration: plugin assumes one Netlify token for all repos; Kueue uses a separate Netlify project (`kubernetes-sigs-kueue`) and may need per-repository credentials
- **2026-07-27** — mimowo (Kueue) confirmed the "stuck" Netlify issue affects Kueue too, and suggested `/retest` could short-circuit for Netlify success without generating traffic, keeping it usable as-is
- **2026-07-27** — Caesarsage committed to concrete follow-up direction:
  - Make `/retest` a quiet, best-effort path; only the explicit command gets user-facing responses
  - Rename the explicit command to `/netlify-rebuild` (converges with petr-muller's earlier naming suggestion)
  - Review and implement pagination (Netlify supports `page`/`per_page` and `Link` headers)
  - Kueue support deferred until a decision is made on shared vs. per-repo credentials; explicitly keeping this PR scoped to generic plugin behavior first
  - Declined to expand `main.go` hot-reload handling in this PR, deferring it to follow-up (COMMENTED review, no code change)

No code changes (SHA unchanged at `35fd83923`). This activity substantially reinforces (and adds detail to) three existing findings — `/retest` overlap, `ListDeploys` pagination, and config hot-reload — and surfaces a new one below.

## CI Status (as of 2026-06-02)

All checks passing:
- `pull-prow-unit-test`: SUCCESS
- `pull-prow-integration`: SUCCESS
- `pull-prow-image-build-test`: SUCCESS
- `pull-prow-verify-lint`: SUCCESS
- `pull-prow-unit-test-race-detector-nonblocking`: SUCCESS
- `EasyCLA`: SUCCESS
- `deploy/netlify`: SUCCESS
- `tide`: PENDING (awaiting lgtm/approve labels)

## Review Checklist

### Converging concerns (3/3 or 2/3 reviewers)
- [ ] ListDeploys pagination: add `per_page=100` or branch filter to avoid false negatives
- [ ] /retest overlap: add code comment on `commandRe` explaining intentional overlap; consider silent no-ops
- [ ] Silent API failures: post user-facing comment when Netlify API calls fail
- [ ] Remove redundant flagutil import

### Moderate — request fix
- [ ] Remove dead netlifyAPIURL validation or remove the default
- [ ] Fix goroutine logging: use request-scoped logger `l` at Error/Warn level
- [ ] Rename misleading test
- [ ] Log discarded webhook validation error at Debug level

### Minor — suggest
- [ ] Bound io.ReadAll on error responses with LimitReader
- [ ] Add server-level tests for edge paths and dry-run
- [ ] Validate config keys match org/repo format
- [ ] Consider config hot-reload for future

### Verify
- [ ] Tests pass: `go test ./cmd/external-plugins/netlify-preview/...`
- [ ] Build succeeds: `go build ./cmd/external-plugins/netlify-preview/`
- [ ] No lint issues

---

## Bottom Line

Well-structured plugin with clean code, good test coverage, and correct reuse of Prow's trust infrastructure. Three independent reviewers (code quality, maintainability, deployment risk) all returned positive assessments. The four converging concerns — `ListDeploys` pagination, `/retest` overlap documentation, silent API failures, and the duplicate import — are the highest-confidence items. None are blockers, but the pagination gap is the most impactful and should be addressed before or immediately after merge.
