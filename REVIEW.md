---
pr: kubernetes-sigs/prow#732
title: "docs: add hook component documentation"
head_sha: ab2db2738583a88b0796c82dfb127cf06da798ca
base: main
reviewed_at: 2026-05-27T12:47:51Z
verdict: request-changes
---

## What this PR does

Replaces the hook component placeholder page (`site/content/en/docs/components/core/hook.md`) with full documentation covering: how hook processes webhooks step-by-step, all seven supported event types, GitHub webhook and HMAC secret configuration, `plugins.yaml` structure with org/repo scoping and `excluded_repos`, external plugin config, CLI flags with defaults, HTTP endpoints, and a troubleshooting section.

## Findings

### [blocking] Health endpoint is misdocumented — will break probe configuration
- where: `site/content/en/docs/components/core/hook.md:149-152`
- concern: `/` on port 8888 is documented as "Health check endpoint (returns 200 OK)" but it is a legacy stub marked in source with `// TODO remove this health endpoint when the migration to health endpoint is done`. The real health infrastructure is `/healthz` (liveness) and `/healthz/ready` (readiness) on the health port (default 8081), served by `pjutil.NewHealthOnPort`. Operators configuring Kubernetes probes from this doc will target the wrong port and endpoint. Flagged independently by Code Quality and Deployment Risk reviewers.
- excerpt: |
    - `/hook` - Webhook receiver endpoint (configurable via `--webhook-path`)
    - `/plugin-help` - Returns help information about enabled plugins
    - `/` - Health check endpoint (returns 200 OK)

### [should-fix] GitHub authentication flags entirely absent from CLI section
- where: `site/content/en/docs/components/core/hook.md:130-143`
- concern: `--github-token-path` (personal access token) and `--github-app-id`/`--github-app-private-key-path` (GitHub App auth) are not listed. Hook cannot call the GitHub API — and plugins cannot respond to events — without one of these. An operator following only this doc will have a Hook that receives webhooks but cannot act on them.
- excerpt: |
    Common flags for Hook:
    
    - `--config-path`: Path to [Prow config file](/docs/config/)
    - `--plugin-config`: Path to plugin config (default: `/etc/plugins/plugins.yaml`)
    - `--hmac-secret-file`: Path to HMAC secret file (default: `/etc/webhook/hmac`)

### [should-fix] No mention of ghproxy or --github-endpoint
- where: `site/content/en/docs/components/core/hook.md:130-143`
- concern: The Prow codebase itself warns that ghproxy is a required production component. Without `--github-endpoint` pointing at a ghproxy instance, operators will hit GitHub API rate limits. Neither the flag nor ghproxy are mentioned anywhere in the doc.
- excerpt: |
    Common flags for Hook:
    [--github-endpoint absent]

### [should-fix] --dry-run footgun is undersold
- where: `site/content/en/docs/components/core/hook.md:143`
- concern: The `--dry-run=true` default means Hook will silently accept webhooks, appear to work in logs, and produce zero real effects (no ProwJob creation, no GitHub comments, no status updates). The doc mentions this only as a single sentence at the bottom of the flags section. It should be a prominent callout in the Configuration section.
- excerpt: |
    Production deployments must set `--dry-run=false`.

### [nit] Stale GitHub webhook docs URL
- where: `site/content/en/docs/components/core/hook.md:8`
- concern: The linked URL uses GitHub's deprecated `/en/developers/webhooks-and-events/` path. Current canonical URL is `https://docs.github.com/en/webhooks/about-webhooks`. Old URL still redirects but is not the canonical form.
- excerpt: |
    [GitHub webhooks](https://docs.github.com/en/developers/webhooks-and-events/webhooks/about-webhooks)

### [nit] See Also section omits external plugins
- where: `site/content/en/docs/components/core/hook.md:169-171`
- concern: The intro links to `/docs/components/external-plugins/` and there is a full External Plugins configuration subsection, but See Also only lists plugins. The directory exists.
- excerpt: |
    ## See Also
    
    - [Plugins Documentation](/docs/components/plugins/)

### [nit] Hardcoded flag defaults will drift silently
- where: `site/content/en/docs/components/core/hook.md:134-141`
- concern: Six defaults are hardcoded in the flags table (ports, paths, durations) with no automated check against source. More durable to omit and note "run `hook --help` for current defaults", keeping only the operationally significant `--dry-run` prose note which is already present.
- excerpt: |
    - `--port`: Port to listen on (default: `8888`)
    - `--grace-period`: Duration to handle events on shutdown (default: `180s`)

### [nit] --slack-token-file listed without context
- where: `site/content/en/docs/components/core/hook.md:141`
- concern: This flag exists for the Slack plugin specifically, not for Slack reporting (that is Crier's responsibility). Without that context, operators may think Hook is a Slack reporter.
- excerpt: |
    - `--slack-token-file`: Path to Slack token file (optional)

### [nit] Event types list is a second copy of ground truth
- where: `site/content/en/docs/components/core/hook.md:24-33`
- concern: Accurate now but will silently go stale as new event handlers are added. A note pointing to `pkg/hook/server.go` as source of truth would improve durability.
- excerpt: |
    Hook handles the following GitHub webhook events with internal plugins:

## Checked

- All seven event type strings (`issues`, `issue_comment`, `pull_request`, `pull_request_review`, `pull_request_review_comment`, `push`, `status`) match the `case` labels in `pkg/hook/server.go`'s `demuxEvent` switch exactly.
- All documented CLI flags and defaults (`--webhook-path=/hook`, `--port=8888`, `--dry-run=true`, `--grace-period=180s`, `--hmac-secret-file=/etc/webhook/hmac`, `--plugin-config=/etc/plugins/plugins.yaml`) verified correct against `cmd/hook/main.go`.
- Plugin config YAML examples structurally valid and match the `OrgPlugins` struct (`plugins`, `excluded_repos` fields).
- External plugin YAML accurately reflects the `ExternalPlugin` struct fields (`name`, `endpoint`, `events`).
- HMAC Secret YAML: key `hmac` + mount at `/etc/webhook` produces `/etc/webhook/hmac` matching the flag default.
- "How Hook Works" six-step flow correctly captures `ServeHTTP` → `demuxEvent` → `demuxExternal` sequence.
- Internal links `/docs/components/plugins/`, `/docs/components/external-plugins/`, `/docs/config/` all resolve to existing directories/files.
- "All event types can be forwarded to external plugins" is accurate per the `default` case in `demuxEvent` routing to `demuxExternal`.

## Open questions

- Should the Endpoints section keep `/` documented at all, or just drop it? If kept, it needs a clear "legacy only, do not use for probes" note.
- Is there a project convention on whether CLI flag sections should be exhaustive or explicitly partial ("common flags")? If partial is intentional, a "see `hook --help` for all flags" note would help readers.
- Would you like a follow-up PR to add GitHub auth flags and a ghproxy reference, or should those go in this PR?
