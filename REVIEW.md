---
pr: kubernetes-sigs/prow#732
title: "docs: add hook component documentation"
head_sha: ab2db2738583a88b0796c82dfb127cf06da798ca
base: main
reviewed_at: 2026-05-27T12:47:51Z
verdict: request-changes
gate:
  decision: hold
  gated_at: 2026-06-01T18:16:02Z
  gated_head_sha: ab2db2738583a88b0796c82dfb127cf06da798ca
  reviewed_head_sha: ab2db2738583a88b0796c82dfb127cf06da798ca
---

## Gate

**Decision: HOLD**

The blocking finding from the local review ("Health endpoint is misdocumented") is not addressed — the SHA is unchanged and `hook.md:151` still reads `- \`/\` - Health check endpoint (returns 200 OK)`, which is factually wrong (legacy stub, TODO-remove comment in source; real endpoints are `/healthz`/`/healthz/ready` on port 8081). This is the only code-level gate. Additionally, the PR lacks a required `/approve` from an OWNERS approver: @Prucek's APPROVED is an LGTM, not an approve — the bot explicitly states the PR is NOT APPROVED and is waiting on @matthyx (or any other approver in OWNERS: petr-muller, matthyx, droslean, krzyzacy, smg247).

**No independent merge risk** — this is a docs-only PR with no code changes. No API surface, configuration schema, or behavioral changes.

**Gating items:**
- `site/content/en/docs/components/core/hook.md:151` — blocking finding not addressed: `/` still documented as health check endpoint; must be replaced with `/healthz`/`/healthz/ready` on health port (8081). Source: REVIEW.md `[blocking]`.
- Process: PR is NOT APPROVED per k8s-ci-robot; needs `/approve` from an OWNERS approver (@matthyx or equivalent). @Prucek approved (LGTM) but is not in the approvers list.

**Unblocks on:** Fix to `hook.md:151` and `/approve` from a listed OWNERS approver.

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

## Followups

### [1] docs: Create operator-facing GitHub authentication and ghproxy setup page
- category: docs
- necessity: should
- where: new file under `site/content/en/docs/` (exact path TBD; see scope)

```
In kubernetes-sigs/prow, following PR #732 ("docs: add hook component documentation"), create an operator-facing documentation page covering how Prow components authenticate to GitHub and how to deploy ghproxy.

Context: PR #732 added hook.md but omitted GitHub authentication flags and ghproxy from the hook docs. The review explicitly deferred these to a follow-up because they apply across multiple components (hook, tide, crier, deck) and deserve a single linkable page rather than being duplicated in each component doc. The existing site/content/en/docs/github.md is developer-focused (how to use the Go client library); the existing site/content/en/docs/ghproxy/_index.md covers ghproxy internals. What is missing is an operator-facing page about setting up GitHub credentials and routing API calls through ghproxy.

Task:
1. Determine the right location for a new operator-facing page. Look at site/content/en/docs/ structure — a file like `github-integration.md` or a section like `getting-started-github.md` is likely appropriate. Check `getting-started-deploy.md` to avoid duplicating content already there.
2. Create the page covering:
   a. GitHub token authentication: `--github-token-path`, creating and mounting a Kubernetes Secret for the token, which components use it.
   b. GitHub App authentication: `--github-app-id` + `--github-app-private-key-path`, when to prefer App auth over token auth, creating and mounting the credentials.
   c. ghproxy: what it is, why it's needed (rate limiting), `--github-endpoint` flag, reference to the existing ghproxy docs at /docs/ghproxy/.
   d. A short note on which Prow components need GitHub credentials (hook, tide, crier, deck at minimum).
3. Add a cross-reference link from `site/content/en/docs/components/core/hook.md`'s CLI Flags section to this new page, replacing any redundant inline content.
4. Optionally add cross-references from crier.md and the tide docs if they exist and are non-placeholder.

Do not duplicate Kubernetes Secret YAML examples in hook.md — link to the new page instead.
Do not modify the ghproxy docs themselves.

Acceptance criteria: The new page builds without Hugo errors. hook.md links to it from the CLI Flags section. The page covers all three areas (token, App, ghproxy). No duplicate credential examples in component docs.

Out of scope: Changes to the Go client library docs (github.md), changes to ghproxy internals documentation, adding GitHub App credential rotation or advanced token management.
```

### [2] docs: Elevate --dry-run=true warning in hook.md
- category: docs
- necessity: should
- where: `site/content/en/docs/components/core/hook.md:38-85` (Configuration section)

```
In kubernetes-sigs/prow, following PR #732 ("docs: add hook component documentation"), improve the --dry-run warning in site/content/en/docs/components/core/hook.md.

Context: The PR added documentation for the Hook component but placed the --dry-run warning only as a single sentence at the bottom of the CLI Flags section ("Production deployments must set --dry-run=false."). The --dry-run=true default is the most common footgun for new Prow operators: Hook silently accepts webhooks, logs events, and takes zero real actions (no ProwJob creation, no GitHub comments, no status updates). Operators often wire up webhooks, see traffic in logs, and spend significant time debugging why nothing happens.

Task:
1. Add a prominent callout or bold warning block near the top of the Configuration section in hook.md (before or at the top of the "GitHub Webhook Setup" or "HMAC Secret" subsection). The warning should state:
   - Hook defaults to --dry-run=true
   - In dry-run mode: webhooks are received and validated, but no GitHub API mutations are made and no ProwJobs are created
   - Production deployments must set --dry-run=false
2. Remove the redundant single sentence ("Production deployments must set --dry-run=false.") at the bottom of the CLI Flags section, since the warning is now more prominent above.
3. The Hugo docs site uses Markdown — if there is an established admonition/callout shortcode (e.g. {{% alert %}}, {{< warning >}}, or similar), use it. Otherwise use a Markdown blockquote with bold prefix: > **Warning:** ...

Acceptance criteria: A reader scanning the Configuration section sees the dry-run warning before setting up their first configuration item. The duplicate sentence in the CLI Flags section is removed. The page builds without Hugo errors.

Out of scope: Changes to other component docs, changes to the actual --dry-run behavior, adding dry-run to the troubleshooting section.
```

### [3] docs: Fix deprecated GitHub webhook docs URL
- category: docs
- necessity: could
- where: `site/content/en/docs/components/core/hook.md:8`

```
In kubernetes-sigs/prow, following PR #732 ("docs: add hook component documentation"), fix a stale external link in site/content/en/docs/components/core/hook.md.

Context: Line 8 of hook.md links to GitHub's webhook documentation using the old URL path:
  https://docs.github.com/en/developers/webhooks-and-events/webhooks/about-webhooks
GitHub has migrated this content to:
  https://docs.github.com/en/webhooks/about-webhooks
The old URL still redirects, so this is not broken — it is a link hygiene fix.

Task: Replace the URL in the intro paragraph on line 8. The link text "GitHub webhooks" should remain unchanged; only the URL changes.

Old: https://docs.github.com/en/developers/webhooks-and-events/webhooks/about-webhooks
New: https://docs.github.com/en/webhooks/about-webhooks

Acceptance criteria: The URL is updated. The page builds without errors. No other changes to the file.

Out of scope: Any other content changes to hook.md.
```

### [4] docs: Add external plugins to See Also section in hook.md
- category: docs
- necessity: could
- where: `site/content/en/docs/components/core/hook.md:169-171`

```
In kubernetes-sigs/prow, following PR #732 ("docs: add hook component documentation"), add a missing link to the See Also section of site/content/en/docs/components/core/hook.md.

Context: The hook.md intro paragraph and the Configuration section both reference external plugins with a link to /docs/components/external-plugins/, but the See Also section at the bottom of the page only lists the plugins page. This is a minor navigation consistency gap.

Task: Add the following entry to the See Also list at the bottom of hook.md:
  - [External Plugins Documentation](/docs/components/external-plugins/)

The existing entry [Plugins Documentation](/docs/components/plugins/) should remain unchanged.

Acceptance criteria: The See Also section contains both the plugins and external plugins links. The page builds without errors. No other changes.

Out of scope: Any other content changes to hook.md.
```

### [5] docs: Add --help note to CLI flags section in hook.md
- category: docs
- necessity: could
- where: `site/content/en/docs/components/core/hook.md:130-143`

```
In kubernetes-sigs/prow, following PR #732 ("docs: add hook component documentation"), add a note to the CLI Flags section of site/content/en/docs/components/core/hook.md indicating that the table lists only common flags and that defaults may drift.

Context: The CLI Flags section hardcodes default values for six flags copied from cmd/hook/main.go. These will silently drift as the project evolves. The existing defaults are accurate, so there is no need to remove them — but a note pointing operators to --help prevents confusion when they encounter discrepancies.

Task: Add a single sentence to the CLI Flags section — either as an introductory sentence or a trailing note — with approximately this content:
  "Run `hook --help` for the full flag list and to verify current defaults."

The existing flag entries and their default values should remain unchanged.

Acceptance criteria: The sentence is present in the CLI Flags section. The page builds without errors. Existing flag entries are unmodified.

Out of scope: Removing or changing any existing flag entries or defaults.
```

### [6] docs: Add plugin-specific flags pattern note to hook.md CLI section
- category: docs
- necessity: could
- where: `site/content/en/docs/components/core/hook.md:130-143`

```
In kubernetes-sigs/prow, following PR #732 ("docs: add hook component documentation"), add documentation to site/content/en/docs/components/core/hook.md explaining that some plugins require additional flags or credentials beyond Hook's core flags.

Context: The CLI Flags section lists --slack-token-file as simply "optional" with no explanation of why it exists or when it is needed. More broadly, the pattern — that individual plugins may require their own credentials or flags mounted into Hook's deployment — is undocumented and is a common source of confusion for operators enabling non-trivial plugins.

Task: Add a short note or subsection to the CLI Flags section (e.g. "Plugin-specific flags") explaining:
  - Some plugins require additional flags or credentials to be passed to Hook at startup
  - These are typically mounted as Kubernetes Secrets and referenced via flags
  - Example: --slack-token-file is needed only when using the Slack plugin to post notifications
  - Each plugin's documentation lists its specific flag and credential requirements

The note should be 2-4 sentences or a short bullet list. The existing --slack-token-file flag entry should remain in the flag list (it can be updated to reference the new note).

Acceptance criteria: The pattern is documented. --slack-token-file has enough context to understand when it applies. The page builds without errors.

Out of scope: Documenting the specific flags for every plugin, changes to any plugin documentation pages.
```

## Open questions

- Should the Endpoints section keep `/` documented at all, or just drop it? If kept, it needs a clear "legacy only, do not use for probes" note.
- Is there a project convention on whether CLI flag sections should be exhaustive or explicitly partial ("common flags")? If partial is intentional, a "see `hook --help` for all flags" note would help readers.
- Would you like a follow-up PR to add GitHub auth flags and a ghproxy reference, or should those go in this PR?
