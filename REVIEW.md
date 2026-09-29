---
pr: "kubernetes-sigs/prow#708"
title: "external-plugins: add netlify-preview plugin to retry deploy previews"
head_sha: e8849971aad14f136ab4b965d06a9fe679b9115b
base: main
reviewed_at: 2026-09-29T17:56:49Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-09-29T17:57:34Z
  gated_head_sha: e8849971aad14f136ab4b965d06a9fe679b9115b
  reviewed_head_sha: e8849971aad14f136ab4b965d06a9fe679b9115b
---

# Review of PR #708

## Gate

**Do not merge.** The PR head is unchanged since the saved review, and both blocking findings still exist in the current code. Fixing them is required before another gate pass; the remaining should-fix findings also need a disposition. No new compatibility risk for existing Prow deployments was found.

### Gating list

- **Unaddressed, blocks merge — `REVIEW.md`:** The client follows an unvalidated pagination URL and adds the Netlify token ([`netlify/client.go:80-99`](cmd/external-plugins/netlify-preview/netlify/client.go)). Restrict authenticated requests to the configured HTTPS API origin.
- **Unaddressed, blocks merge — `REVIEW.md`:** An untrusted comment author can force a rebuild when the PR author or `ok-to-test` label makes the PR trusted ([`server.go:197-207`](cmd/external-plugins/netlify-preview/server.go)). Require a trusted author for `/netlify-rebuild`.

### Other findings still open

- **Unaddressed — `REVIEW.md`:** The active-state guard covers only `building` and `enqueued` (`plugin/plugin.go:82-86`); case-insensitive parsing and first-command-only dispatch can ignore an explicit rebuild (`plugin/plugin.go:37-45`); and Netlify API failures leave an explicit rebuild unanswered (`server.go:154-159,177-181`). Resolve or explicitly disposition these behavior findings before merge.
- **Unaddressed — `REVIEW.md`:** The docs omit the mapping restart and `--dry-run=false` requirement (`main.go:80,119`); asynchronous errors lack PR context (`server.go:86-89`); and Netlify operations have no concurrency limit (`server.go:86-90`). These are operational should-fix findings for the new plugin.
- **Borderline — `@petr-muller`:** The request for live mapping reload was explicitly deferred as follow-up in the PR discussion. That deferral is acceptable for this opt-in plugin if the restart requirement is documented.

### Independent merge risk

- The diff adds one image registration, a new external-plugin directory, and a new documentation page. It does not change existing exported packages, CRDs, configuration fields, flags, or ProwJob behavior; existing deployments have no direct compatibility impact.
- Operators who enable the new binary must provide its config and credentials and set `--dry-run=false` for actual retries. The unresolved authorization and request-volume issues affect those opt-in deployments and their shared Netlify account.
- Earlier inline feedback on command naming, quiet `/retest`, and deploy pagination was addressed in the current head; it does not add another gate.

## Verdict

**Request changes.** Authenticated Netlify pagination can follow a cross-origin URL, and an untrusted commenter can repeatedly force rebuilds on a trusted PR. The existing command-handling and retry findings below also warrant attention. The later [PR discussion](https://github.com/kubernetes-sigs/prow/pull/708#issuecomment-5097599488) establishes `/netlify-rebuild` and a quiet `/retest` as the intended behavior, superseding the older PR description.

## What this PR does

- Adds an external Prow plugin that responds to `/retest` and `/netlify-rebuild` on pull requests.
- Uses Prow's trigger trust rules, then finds the latest matching Netlify deploy preview for the PR.
- Lists up to ten pages of previews and retries eligible deploys through the Netlify API.
- Adds plugin configuration, image registration, documentation, and unit tests.

## Findings

### [blocking] Restrict authenticated pagination to the Netlify API origin
- where: `cmd/external-plugins/netlify-preview/netlify/client.go:80-99`
- concern: `nextLink` can supply an arbitrary absolute URL, and `listDeploysPage` adds the Netlify bearer token before requesting it. If a `Link` header contains a cross-origin URL, the token is disclosed to that origin. Validate HTTPS and the configured API origin before following a pagination link.
- excerpt: |
    pageDeploys, next, err := c.listDeploysPage(ctx, nextURL)
    // ...
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, deploysURL, nil)
    // ...
    c.authorize(req)

### [blocking] Require a trusted commenter for force rebuilds
- where: `cmd/external-plugins/netlify-preview/server.go:197-207`
- concern: When the comment author fails `TrustedUser`, this fallback still accepts a trusted PR author or an `ok-to-test` label. Thus any commenter can repeatedly invoke `/netlify-rebuild` on that PR, contrary to the plugin documentation's comment-author rule. Gate the force command on the comment author's trust.
- excerpt: |
    if trustedResponse.IsTrusted {
        return true, nil
    }
    _, trusted, err := trigger.TrustedPullRequest(s.ghc, triggerConfig, issueAuthor, org, repo, number, nil)
    return trusted, err

### [should-fix] Skip retries for every active deploy state
- where: `cmd/external-plugins/netlify-preview/plugin/plugin.go:82-86`
- concern: Netlify can report an in-progress deploy as `retrying`, `uploading`, `preparing`, or `processing`, but this guard recognizes only `building` and `enqueued`. A `/netlify-rebuild` comment during one of those states reaches `RetryDeploy` again, causing a duplicate request or an API rejection. Classify all active states before deciding to retry; the [Netlify API state list](https://open-api.netlify.com/) includes these values.
- excerpt: |
    if preview.State == "building" || preview.State == "enqueued" {
        return Decision{Action: ActionAlreadyRunning}
    }
    if command == NetlifyRebuildCommand {
        return Decision{Action: ActionRetry, ShouldRetry: true}
    }

### [should-fix] Make accepted command casing consistent with dispatch
- where: `cmd/external-plugins/netlify-preview/plugin/plugin.go:37-45`
- concern: The case-insensitive regex accepts `/NETLIFY-REBUILD` but returns that spelling unchanged. It does not equal `NetlifyRebuildCommand` later, so a ready preview gets no retry or reply. The same regex accepts `/ReTest`, while Prow's trigger uses a case-sensitive `/retest` regex, so Netlify can retry without CI. Either reject mixed-case commands or normalize them consistently with the trigger.
- excerpt: |
    var commandRe = regexp.MustCompile(`(?mi)^/(retest|netlify-rebuild)\s*$`)

    func ParseCommand(body string) (Command, bool) {
        body = markdown.DropCodeBlock(body)
        matches := commandRe.FindStringSubmatch(body)
        if len(matches) != 2 {
            return "", false
        }
        return Command(matches[1]), true
    }

### [should-fix] Honor the rebuild command when a comment contains both commands
- where: `cmd/external-plugins/netlify-preview/plugin/plugin.go:39-45`
- concern: `FindStringSubmatch` returns only the first command in a comment. For `/retest` followed by `/netlify-rebuild`, the plugin chooses `/retest`; when the preview is ready, it silently does nothing despite the explicit rebuild request. Parse all command lines or define and enforce an unambiguous priority.
- excerpt: |
    matches := commandRe.FindStringSubmatch(body)
    if len(matches) != 2 {
        return "", false
    }
    return Command(matches[1]), true

### [should-fix] Tell the user when the Netlify API rejects a rebuild
- where: `cmd/external-plugins/netlify-preview/server.go:177-181`
- concern: `/netlify-rebuild` replies on successful retries and no-op decisions, but a failed `RetryDeploy` returns an error without posting a reply. A failed `ForEachDeployPage` at lines 154-159 has the same result. For a rate limit or API outage, the commenter sees a command with no response and cannot tell whether a retry happened.
- excerpt: |
    if !s.dryRun {
        if err := s.netlifyClient.RetryDeploy(ctx, preview.ID); err != nil {
            return err
        }
    }

### [should-fix] Document the restart and production flag
- where: `cmd/external-plugins/netlify-preview/main.go:80-83`
- concern: The site mapping is loaded only at startup, so changing it requires a restart. The binary also defaults to `--dry-run=true`, so an operator following the new plugin page will not trigger real retries until setting `--dry-run=false`. Document both steps in the deployment instructions.
- excerpt: |
    fs.BoolVar(&o.dryRun, "dry-run", true, "Dry run for testing. Uses API tokens but does not mutate.")
    fs.StringVar(&o.configPath, "config-path", "/etc/netlify-preview/config.yaml", "Path to the netlify-preview plugin config file.")

### [should-fix] Include PR context in asynchronous failures
- where: `cmd/external-plugins/netlify-preview/server.go:86-89`
- concern: The failure logger uses the original entry, which has only event type and GUID. Repository, PR number, and command fields are added to a separate entry inside `handleIssueComment`, so API failures are hard to trace back to the request. Carry those fields into the error log or wrap returned errors with them.
- excerpt: |
    go func() {
        if err := s.handleIssueComment(l, ic); err != nil {
            s.log.WithError(err).WithFields(l.Data).Info("Failed to handle issue comment.")
        }
    }()

### [should-fix] Bound concurrent Netlify operations
- where: `cmd/external-plugins/netlify-preview/server.go:86-90`
- concern: Each actionable comment starts an unrestricted goroutine and can list up to ten pages. A burst of comments can consume shared Netlify API capacity. Bound concurrent operations or add a per-PR cooldown.
- excerpt: |
    go func() {
        if err := s.handleIssueComment(l, ic); err != nil {
            s.log.WithError(err).WithFields(l.Data).Info("Failed to handle issue comment.")
        }
    }()

## Checked

- Compared the PR head `e8849971aad14f136ab4b965d06a9fe679b9115b` with its merge base on `upstream/main` and inspected all changed production code, tests, docs, and image registration.
- Confirmed webhook signature validation uses Prow's `github.ValidateWebhook` and trust checks use the trigger plugin's helpers.
- Confirmed the later PR discussion intentionally replaced `/rebuild-preview` with `/netlify-rebuild` and made `/retest` quiet on a ready preview; those are not current behavior defects.
- Existing Prow deployments are unaffected until they opt in to this external plugin; no existing configuration fields or ProwJob behavior changed.
- `go test ./cmd/external-plugins/netlify-preview/...` passed.

## Open questions

- None.
