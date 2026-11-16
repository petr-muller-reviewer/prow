---
pr: kubernetes-sigs/prow#1037
title: "peribolos: support org-level repo defaults"
head_sha: 2930af9fa960fe74d13efd67837b3191f9abedc7
base: main
reviewed_at: 2026-11-16T23:53:49Z
verdict: request-changes
---

# Review

## Verdict

Request changes: the message-only squash path sends a payload GitHub's API description says is invalid. Everything else in the PR is sound.

The feature is well scoped and tested: defaults are opt-in, the flat YAML/JSON format is preserved, archived and unlisted repos are handled, and the docs cover the surprising cases. The `RepoRequest.Defined()` fix is the reason message-only squash changes now reach the API — but `newRepoUpdateRequest` and `newRepoCreateRequest` then omit `squash_merge_commit_title` while sending `squash_merge_commit_message`, and GitHub's published OpenAPI description marks the title "Required when using `squash_merge_commit_message`". That needs to be fixed (send the effective title, or reject locally) before the exact case the PR targets can be trusted.

## What this PR does

- Adds an optional org-level `repo_defaults` block sharing nine repository settings; listed repositories inherit unset values and keep explicit overrides, including `false`.
- Extracts those fields into `org.RepoSettings` and embeds it in `org.Repo`, keeping the YAML/JSON repository format flat and legacy `private` parsing intact; `repo_defaults` uses a separate alias type with strict key checking.
- Applies defaults only to listed repositories on create and update, ignores them for repos archived on GitHub when `archived` is unset, and still manages explicit per-repository settings.
- Makes `RepoRequest.Defined()` recognize the squash title/message, validates the effective squash pair before create/update, and omits squash fields when squash merging will be disabled.
- Documents scope, per-repo overrides, archived-repo behavior, dump limitations and binary-before-config rollout; the final commit keeps Go 1.26.4 and drops the unrelated Jira, rifle, lint and module changes.

## Findings

### [blocking] Message-only squash requests omit the title GitHub requires

- where: `cmd/peribolos/main.go:1379-1380`, `cmd/peribolos/main.go:1288-1289`
- concern: When only the squash message differs, the request contains `squash_merge_commit_message` and no `squash_merge_commit_title`. GitHub's OpenAPI description for `PATCH /repos/{owner}/{repo}`, `POST /orgs/{org}/repos` and `POST /user/repos` says of `squash_merge_commit_title`: "Required when using `squash_merge_commit_message`." The new `Defined()` behaviour is what activates these requests, so the message-only case this PR is meant to fix is the one at risk of a 422. Send the effective current title alongside the message on update (falling back to `COMMIT_OR_PR_TITLE` when the observed title is empty) and `COMMIT_OR_PR_TITLE` on create, or reject a message without a title locally with a clear error. The tests at `cmd/peribolos/repos_test.go:569-592` assert the current title-less payloads and would need updating.
- excerpt: |
    			SquashMergeCommitTitle:   setIfChanged(current.SquashMergeCommitTitle, repo.SquashMergeCommitTitle),
    			SquashMergeCommitMessage: setIfChanged(current.SquashMergeCommitMessage, repo.SquashMergeCommitMessage),

### [should-fix] WithDefaults can silently miss a new RepoSettings field

- where: `pkg/config/org/org.go:146-178`
- concern: `RepoDefaults` is an alias of `RepoSettings` (`org.go:130`), so a field added to `RepoSettings` is automatically accepted by the strict decoder but is not inherited unless someone also edits this hand-written list. That is exactly the silent-misconfiguration class the strict decode is meant to prevent, and neither the compiler nor the current fixtures catch it. A reflect-based parity test, or a merge implementation that cannot drift, would close the gap.
- excerpt: |
    func (r Repo) WithDefaults(defaults *RepoDefaults) Repo {
    	if defaults == nil {
    		return r
    	}
    	if r.AllowSquashMerge == nil {
    		r.AllowSquashMerge = defaults.AllowSquashMerge
    	}

### [nit] Importer migration note lives only in the PR description

- where: `pkg/config/org/org.go:63-64`
- concern: Embedding `RepoSettings` is a source break for external Go consumers using keyed literals with promoted fields. The PR description gives the nested form, but the package doc and site docs do not, so a downstream compile error will not explain itself. Add a sentence on `Repo` showing the nested literal form.
- excerpt: |
    type Repo struct {
    	RepoSettings

### [nit] Archived-repo fallback is implicit

- where: `cmd/peribolos/main.go:1494-1503`
- concern: Reaching back into `orgConfig.Repos` to "un-apply" defaults works but is easy to misread or break in a refactor that resolves defaults earlier. A named helper or a comment carrying the rationale would localize the rule.
- excerpt: |
    				if wantRepo.Archived == nil {
    					wantRepo = orgConfig.Repos[wantName]
    					if orgConfig.RepoDefaults != nil {
    						repoLogger.Infof("repo %q is archived, ignoring repository defaults", wantName)
    					}
    				}

### [nit] Effective squash settings are duplicated between create and update

- where: `cmd/peribolos/main.go:1292-1296`, `cmd/peribolos/main.go:1391-1394`
- concern: The effective `allow_squash_merge` computation and the "drop title/message when disabled" rule exist twice, and the GitHub enum strings exist a third time in validation (`1347-1352`) and a fourth in the create-path defaults (`1473-1474`). A shared helper plus typed constants would keep the branches consistent and make a future API value one edit. Configured squash settings dropped because squash is disabled are also not logged.
- excerpt: |
    	if !allowSquashMerge {
    		repoUpdate.SquashMergeCommitTitle = nil
    		repoUpdate.SquashMergeCommitMessage = nil
    	}

### [nit] repo_defaults decode errors lack the org name

- where: `pkg/config/org/org.go:133-141`
- concern: The error names the block and field but not the org, so a multi-org config still requires grepping to find the offending block. `UnmarshalJSON` also decodes a single JSON value and ignores trailing data.
- excerpt: |
    func (r *RepoDefaults) UnmarshalJSON(data []byte) error {
    	var settings RepoSettings
    	decoder := json.NewDecoder(bytes.NewReader(data))
    	decoder.DisallowUnknownFields()
    	if err := decoder.Decode(&settings); err != nil {
    		return fmt.Errorf("repo_defaults: %w", err)
    	}

### [question] Can observed squash values be empty?

- where: `cmd/peribolos/main.go:1340-1348`
- concern: Validation treats the values from the repo lookup as always populated. If a GHES version omits them, or returns them empty when squash merging is disabled, a valid single-sided configuration is rejected locally. Should empty observed values be treated as unknown (skip the unspecified half or substitute GitHub's defaults)?
- excerpt: |
    	title, message := current.SquashMergeCommitTitle, current.SquashMergeCommitMessage
    	if request.SquashMergeCommitTitle != nil {
    		title = *request.SquashMergeCommitTitle
    	}
    	if request.SquashMergeCommitMessage != nil {
    		message = *request.SquashMergeCommitMessage
    	}

## Resolved

Earlier feedback on this PR is addressed at `2930af9fa` and is not re-raised above.

- **Go minimum and unrelated changes** (raised by @cblecker): `go.mod` is back to `go 1.26.4`, literals use the nested `RepoSettings` form, and the Jira, rifle, `embedlit` and module changes are gone from the diff.
- **Archived-repo and squash behavior** (raised by @cblecker): defaults are ignored for archived repos with unset archival while explicit settings still surface the read-only error, and the squash pair is validated on create and update with squash fields omitted when disabled.
- **Strict `repo_defaults` decoding and docs** (raised by @cblecker): a separate alias type rejects unsupported keys, and the docs cover null inheritance, the absence of a per-repo opt-out, dump pruning, archived repos and rollout ordering.

## Checked

- Flat JSON/YAML repository configs still parse unchanged (peribolos uses `sigs.k8s.io/yaml`), including legacy `private`, `null` inheritance and explicit `false` overrides; source configs are not mutated.
- Inheritance covers nil/empty defaults, supported fields, overrides, unlisted repositories and `PruneRepoDefaults` for dumps.
- `repo_defaults` rejects unknown keys, `visibility`, `private`, `archived`, typos and wrong types; per-repository parsing stays permissive.
- Create and update paths apply defaults, skip archived repos appropriately, omit squash fields when squash will be disabled, and `Defined()` now recognizes squash-only changes.
- Deployment: no new dependencies, API calls, permissions or per-request work beyond the existing O(#repos) loop; the feature is inert without `repo_defaults`; rollback leaves applied repo settings in place.
- Verified the OpenAPI wording for `squash_merge_commit_title` on `POST /orgs/{org}/repos`, `PATCH /repos/{owner}/{repo}` and `POST /user/repos`.
- Tests were not independently re-run here; the reviewers ran `go test ./pkg/config/org ./pkg/github ./cmd/peribolos` at `2930af9fa` and reported it passing. No live GitHub write was performed.

## Open questions

- Will GitHub.com (and which GHES versions) actually reject a PATCH/POST that sets only `squash_merge_commit_message`? A check against a scratch repo would settle it.
- Do any supported GHES versions omit `squash_merge_commit_title`/`squash_merge_commit_message` from repo responses?

