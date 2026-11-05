---
pr: kubernetes-sigs/prow#1037
title: "peribolos: support org-level repo defaults"
head_sha: 9a62ef160563080b84e47579e2464329edbf7140
base: main
reviewed_at: 2026-11-04T18:00:23Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-11-04T22:56:21Z
  gated_head_sha: 9a62ef160563080b84e47579e2464329edbf7140
  reviewed_head_sha: 9a62ef160563080b84e47579e2464329edbf7140
---

# Review

## Gate

**Verdict: do-not-merge.** The PR head is unchanged (`9a62ef16`), so the review's `[blocking]` create-path finding still stands, and a new review by @cblecker (2026-11-04, `COMMENTED`) adds follow-ups that require code changes. Independently of the reviews, the `Defined()` change makes previously-ignored squash settings take effect on upgrade, which can hard-fail an entire run for configs that never use `repo_defaults`.

The gate is blocked on a small, well-scoped set of changes, not on the design. The inheritance model, strict `repo_defaults` decoding, flat config schema and create/update parity are sound and tested; @cblecker's earlier items (dry-run wording, dump limitations, unknown keys) are addressed. What remains is squash commit title/message validity: GitHub validates the two fields **as a pair**, and this PR sends them independently — defaults resolve field-by-field, so a repo that overrides only one of them can produce a combination GitHub rejects. That breaks `repo_defaults` adopters on create, and now also breaks existing configs that set squash settings which `Defined()` previously dropped. Fixing the pair rule once, at the point the request is built, plus narrowing the archived-repo guard, unblocks the merge.

### Gating list

1. **Squash title/message pair is not validated** — `cmd/peribolos/main.go:1288-1289` (create path) and `cmd/peribolos/main.go:1360-1363` (update fallback). Source: `REVIEW.md` `[blocking]`, confirmed by @cblecker 2026-11-04. `newRepoCreateRequest` forwards `squash_merge_commit_message` with no title, and no code in this head validates the effective pair. @cblecker reproduced 422s for an invalid or empty title, for `PR_BODY` with a `COMMIT_OR_PR_TITLE` repo (on update *and* on create), and for squash values sent while squash merge is disabled. Validate the resolved pair before the request and report a clear error naming the repo and field, and document the valid pairs (`PR_TITLE` with `PR_BODY`/`BLANK`/`COMMIT_MESSAGES`; `COMMIT_OR_PR_TITLE` with `COMMIT_MESSAGES`) — this should replace the `""`-override test that only covers the update path.
2. **Archived-repo guard is too wide** — `cmd/peribolos/main.go:1450-1452`. Source: @cblecker 2026-11-04; supersedes the review's documentation-only `[should-fix]`. The skip fires on `existing.Archived && wantRepo.Archived == nil`, so an archived repo silently drops *explicit* per-repo settings (a new `description`, a `previously` rename) that used to fail the run loudly. Limit the skip to inherited values by computing the delta from `orgConfig.Repos[wantName]`, add a `TestConfigureRepos` case (explicit `description`, unset `archived`, no defaults), and document: "Repositories archived on GitHub do not inherit defaults unless the config sets `archived: false`, which is reported as an unsupported unarchive."
3. **Squash settings are sent when squash merging will be disabled, and the backfill should go** — `pkg/config/org/org.go:119`, `cmd/peribolos/main.go:1360-1363`, `site/content/en/docs/components/cli-tools/peribolos.md:117`. Source: @cblecker 2026-11-04. Omit `squash_merge_commit_title`/`squash_merge_commit_message` from the request whenever squash merge will be off (disabled by this request or already off) — otherwise the PATCH 422s and takes unrelated changes with it. Drop the title backfill in `newRepoUpdateRequest` (@cblecker does not believe GitHub needs it; it is also the source of the empty-title case). Delete "Configs without `repo_defaults` behave as before" from the docs: after `pkg/github/types.go:491-495`, configs that set squash settings without `repo_defaults` now send them, so the claim is false and the upgrade is behavior-changing.

### Merge risk

- **Behavioral, upgrade-time (highest)**: `pkg/github/types.go:491-495` adds the squash title/message to `Defined()`. Any installation whose config already sets these fields — no `repo_defaults` required — starts sending them after upgrade; where that resolves to an invalid pair, the update 422s and aborts the whole run before collaborators and teams are reconciled. Blast radius: every peribolos installation with squash commit settings. Not opt-in, not currently documented, and contradicted by the docs line above.
- **Behavioral**: archived repos with `archived` unset stop syncing in every installation (INFO log, clean exit), changing both drift enforcement and the exit-code signal, not only for adopters of `repo_defaults`.
- **Go API**: embedding `RepoSettings` in the exported `org.Repo` breaks keyed literals such as `org.Repo{HasIssues: ...}` for downstream importers (openshift/ci-tools `private-org-peribolos-sync` reported by @cblecker). In-tree callers are migrated; no runtime impact, but a mechanical fix is required downstream and deserves a release note and a line in the PR description.
- **Config**: `repo_defaults` is opt-in, `nil` defaults are a no-op, and the on-disk shape stays flat (round-trip tested); older binaries ignore the unknown key under non-strict parsing, so a rollback is safe.
- **Rollout**: one default can mutate every listed repo in a single run with no per-repo opt-out, and the dry run reports repo names only, which makes reviewing a defaults rollout hard. No compatibility break, but worth a release note.

Non-gating: `repo_defaults` decode errors omit the owning org (`pkg/config/org/org.go:133-142`); the two hand-maintained field lists (`org.go:146-178`, `types.go:491-495`) still lack a completeness guard; `RepoDefaults` vs `PruneRepoDefaults` naming.

## Verdict

Request changes. The feature is well-scoped, well-tested and backward-compatible, but the create path does not honor the message-only squash contract the update path was fixed for.

The PR adds `repo_defaults` with per-repo overrides, resolves them through a pure `WithDefaults`, skips archived repos unless config sets `archived: false`, and rejects unknown `repo_defaults` keys via a dedicated `RepoDefaults` type. Inheritance, explicit `false`, `null`, create/update parity, unlisted/archived repos and flat JSON/YAML round-trips are covered. The blocker is `newRepoCreateRequest` sending `squash_merge_commit_message` without a title: a message-only default, which the docs list as supported, breaks creation with a 422 unless the update-side backfill is mirrored. The remaining items are non-blocking hardening and documentation gaps.

## What this PR does

- Adds an optional `repo_defaults` block to each org config, inherited by every repo listed under `repos`.
- Factors defaultable repo fields into a shared `RepoSettings` type embedded in `org.Repo`, keeping the repo JSON/YAML shape flat.
- Resolves defaults in `configureRepos` before creation and update, honoring explicit values (including `false`); `null` means inherit.
- Skips repos GitHub reports as archived unless the config sets `archived: false`, and includes the current squash title on message-only updates.
- Rejects unsupported `repo_defaults` keys through a dedicated strict `RepoDefaults` type and documents the dump/dry-run limitations.

## Findings

### [blocking] Create path sends a squash commit message without a title

- where: `cmd/peribolos/main.go:1288-1289`
- concern: `newRepoCreateRequest` forwards `SquashMergeCommitMessage` without `SquashMergeCommitTitle`. With `repo_defaults` supplying a message-only default (a configuration the docs list as supported), creating a listed repo sends a create request missing the title and fails with a 422. This is the creation-side counterpart of the update fix at `main.go:1360-1363`. Mirror the backfill (default the title to `PR_TITLE`) or reject message-only configs at load time, and add a create-with-message-only test. The code path is CONFIRMED; GitHub enforcing the constraint on create is PLAUSIBLE — the update fix already relies on the same rule, but it was not verified here.
- excerpt: |
    SquashMergeCommitTitle:   definition.SquashMergeCommitTitle,
    SquashMergeCommitMessage: definition.SquashMergeCommitMessage,

### [should-fix] Hand-maintained field lists have no completeness guard

- where: `pkg/config/org/org.go:146-178`
- concern: `WithDefaults` hand-copies the `RepoSettings` field set, and `RepoRequest.Defined()` (`pkg/github/types.go:491-495`) hand-enumerates another. This PR fixes a silent skip caused by an omission in `Defined()`, but adding a field to either struct still compiles and silently leaves it undefaulted or undetected; the tests hard-code the same set. Add a reflection-based completeness test so drift fails loudly.
- excerpt: |
    func (r Repo) WithDefaults(defaults *RepoDefaults) Repo {
        if defaults == nil {
            return r
        }
        if r.AllowSquashMerge == nil {
            r.AllowSquashMerge = defaults.AllowSquashMerge
        }

### [should-fix] Archived-repo skip is not documented

- where: `cmd/peribolos/main.go:1450-1452`
- concern: Archived repos with `archived` unset now stop syncing for every installation, not just those adopting defaults. The behavior is sensible, but `peribolos.md` never mentions it (it only says archival remains per-repository only), so operators lose drift enforcement and the exit-code signal without notice. Add a docs line.
- excerpt: |
    if existing.Archived && (wantRepo.Archived == nil || *wantRepo.Archived) {
        repoLogger.Infof("repo %q is archived, skipping changes", wantName)
        continue
    }

### [should-fix] `repo_defaults` decode errors do not name the owning org

- where: `pkg/config/org/org.go:133-142`
- concern: A strict-decode failure reads `repo_defaults: unknown field "..."` with no org context. `FullConfig` is decoded in one `yaml.Unmarshal`, so an operator with many orgs must search each block. Wrap the error with the org name (or field path) where possible.
- excerpt: |
    func (r *RepoDefaults) UnmarshalJSON(data []byte) error {
        var settings RepoSettings
        decoder := json.NewDecoder(bytes.NewReader(data))
        decoder.DisallowUnknownFields()
        if err := decoder.Decode(&settings); err != nil {
            return fmt.Errorf("repo_defaults: %w", err)
        }

### [nit] Update fallback can send an empty squash title

- where: `cmd/peribolos/main.go:1360-1363`
- concern: When `current.SquashMergeCommitTitle` is empty (e.g. an older GHES that does not return the field), the fallback sends `""`, which is not a valid enum. Return a clear error or skip the update instead of emitting an invalid request.
- excerpt: |
    if repoUpdate.SquashMergeCommitMessage != nil && repoUpdate.SquashMergeCommitTitle == nil {
        repoUpdate.SquashMergeCommitTitle = &current.SquashMergeCommitTitle
    }

### [nit] `RepoDefaults` values are not validated

- where: `pkg/config/org/org.go:133-142`
- concern: Strict decoding validates keys but not values; arbitrary strings (including `""`) pass for the two squash enums and then fail once per listed repo at the API. Validating the enums at load time surfaces one clear error instead of an org-wide failure run.

### [nit] `RepoDefaults` vs `PruneRepoDefaults`, and pointer aliasing

- where: `pkg/config/org/org.go:130`
- concern: `RepoDefaults` (org-level inheritance) and `PruneRepoDefaults` (`pkg/config/org/org.go:326`) are easy to confuse; a cross-reference comment would help. Also, `WithDefaults` copies pointer values, so the returned repo aliases the defaults — safe while config is immutable, but fragile if a caller ever mutates through `*repo.X`.

## Checked

- Config shape: `RepoSettings` is embedded with no JSON tag; flat keys still deserialize and serialize identically, with JSON/YAML round-trip and legacy `private` tests.
- Inheritance: explicit `false` wins over a `true` default, `null` inherits, nil/empty defaults are no-ops, and create/update parity is tested.
- Scope: unlisted repos are untouched and defaults alone manage nothing; explicitly archived repos are skipped.
- Strict decoding: `visibility`, `private`, `archived` and typos are rejected in `repo_defaults`, while per-repo parsing stays permissive.
- Message-only updates include the current title; tests cover configured-title and fallback cases.
- No new API endpoints, permissions, scopes, flags, or background work; the merge is an O(repos) map copy.
- CI on `9a62ef16` is green (unit, race, lint, integration, image build); targeted tests pass.

## Open questions

- Should message-only defaults be allowed? If so, the create path needs the same title backfill as updates; if not, reject them at config load.
- Should `peribolos.md` state that archived repos are skipped unless `archived: false` is set?
- Should the release notes call out that squash settings previously ignored will now take effect on upgrade?
- Should `RepoDefaults` decode errors include the owning org name?
- Should the two squash enums be validated locally so one typo does not fan out into per-repo API failures?
- In dry runs, would logging the effective field delta (instead of just "repo differs") make a defaults rollout reviewable?
