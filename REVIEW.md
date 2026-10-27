---
pr: kubernetes-sigs/prow#1037
title: "peribolos: support org-level repo defaults"
head_sha: 27d8e94d6931f3253dbaa0b10679c6a0d7e09b77
base: main
reviewed_at: 2026-10-27T17:22:55Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-10-27T19:19:43Z
  gated_head_sha: 27d8e94d6931f3253dbaa0b10679c6a0d7e09b77
  reviewed_head_sha: 27d8e94d6931f3253dbaa0b10679c6a0d7e09b77
---

# Review

## Gate

**Decision: hold.** The code is sound and this is not a do-not-merge, but two should-fix items are unaddressed and the `Defined()` behavior change needs a release note before merge.

The PR head is unchanged from the reviewed SHA (`27d8e94d`), so every prior finding still stands. The feature is opt-in and configs without `repo_defaults` keep their current behavior, which rules out a do-not-merge. Holding for three small, scoped items: (1) `repo_defaults` can now produce an update delta for a repo GitHub reports as archived when the config does not declare `archived: true`, and the recommended dry run cannot surface the resulting rejected write because `UpdateRepo` short-circuits in dry-run mode (`pkg/github/client.go:2871-2873`); (2) the new docs claim dumps report individual repo settings, but `dumpOrgConfig` omits the two squash commit fields the example promotes; (3) the `Defined()` fix silently activates previously-inert squash settings on upgrade with no release note.

Gating list:

1. `cmd/peribolos/main.go:1445` — archived-repo guard. Source: this review, `[should-fix]`. Skip repos GitHub reports as archived (or explicitly require `archived: true`) and test the undeclared case.
2. `site/content/en/docs/components/cli-tools/peribolos.md:119` and `cmd/peribolos/main.go:465-481` — reconcile the dump claim: either emit/prune the squash commit fields or narrow the wording. Source: this review, `[should-fix]`.
3. `pkg/github/types.go:491` — call out in the docs/changelog that configs setting `squash_merge_commit_title`/`squash_merge_commit_message` will start applying them on upgrade. Source: this review, `[should-fix]`, deployment-risk pass.

Not gating (follow-up): an exhaustiveness test for the hand-written field list in `WithDefaults` (`pkg/config/org/org.go:129`).

Area-2 merge risk:

- **Go API surface** — embedding `RepoSettings` in the exported `org.Repo` breaks downstream keyed literals that reference promoted fields (`pkg/config/org/org.go:62`). In-tree callers were migrated; no runtime or deployment impact, but release-note it if `pkg/config/org` importers matter.
- **Configuration** — `repo_defaults` is optional and `nil` is a strict no-op (tested). Older peribolos versions ignore it silently because config loading is non-strict (`cmd/peribolos/main.go:200`); this is documented and needs no migration.
- **Behavioral** — `RepoRequest.Defined()` now includes the squash fields (`pkg/github/types.go:491`), so existing configs that set them converge GitHub state on the first run after upgrade, potentially changing squash commit message format. Correct bug fix, but release-note-worthy.
- **Blast radius** — one defaults line can change every listed repo, there is no per-repo opt-out, and dry run only logs "repo differs". It is opt-in and documented, so it does not break existing configs.

## Verdict

Approve. The feature is correct, well-tested and opt-in; the remaining items are non-blocking hardening and documentation consistency.

The inheritance semantics are implemented cleanly: `RepoSettings` is shared between `repo_defaults` and `repos`, `WithDefaults` returns a copy without touching the source config, and explicit `false` wins over a `true` default on both creation and update. Configs without `repo_defaults` keep their current behavior, and `--dump` still emits per-repo settings. Three reviewers converged on the same non-blocking concerns: archived repos are only skipped when the config declares `archived: true`, dumps omit the squash commit fields the new docs promote, and both `WithDefaults` and `Defined` hand-enumerate fields. None of those block merge.

## What this PR does

- Adds an optional `repo_defaults` block to each org config, inherited by every repo listed under `repos`.
- Factors the defaultable repo fields into a shared `RepoSettings` type, embedded in `org.Repo` so the repo JSON/YAML shape stays flat.
- Resolves defaults in `configureRepos` via `Repo.WithDefaults` before both creation and update, honoring explicit values, including `false`; `null` means inherit.
- Fixes `RepoRequest.Defined()` so updates that only change `squash_merge_commit_title` or `squash_merge_commit_message` are no longer silently skipped.
- Documents the feature and adds tests for inheritance, overrides, flat JSON/YAML round-trips, legacy `private` parsing, creation, updates, and unlisted/archived repos.

## Findings

### [should-fix] Archived repos are only skipped when the config declares `archived: true`

- where: `cmd/peribolos/main.go:1444-1450`
- concern: With defaults, a repo that is archived on GitHub but listed without `archived: true` can now get a non-empty delta where it previously had none, so peribolos attempts an update that GitHub rejects as read-only. The new `"archived repo remains untouched"` test covers only the explicitly-declared case.
- excerpt: |
    if existing != nil {
        if existing.Archived {
            if wantRepo.Archived != nil && *wantRepo.Archived {
                repoLogger.Infof("repo %q is archived, skipping changes", wantName)
                continue
            }
        }

### [should-fix] Dumps omit the squash merge commit fields the new docs promote

- where: `cmd/peribolos/main.go:465-481`
- concern: `dumpOrgConfig` never copies `SquashMergeCommitTitle`/`SquashMergeCommitMessage`, so `peribolos.md:119` ("Dumps continue to report individual repository settings") is not true for the two fields the new example recommends. This predates the PR, but the new docs and the `Defined()` fix make it more visible, and dump-then-apply leaves those settings unmanaged. Either emit them (with pruning in `PruneRepoDefaults`) or narrow the doc wording.
- excerpt: |
    repoConfig := org.PruneRepoDefaults(org.Repo{
        RepoSettings: org.RepoSettings{
            HasIssues:           &full.HasIssues,
            HasProjects:         &full.HasProjects,
            HasWiki:             &full.HasWiki,
            AllowMergeCommit:    &full.AllowMergeCommit,
            AllowSquashMerge:    &full.AllowSquashMerge,
            AllowRebaseMerge:    &full.AllowRebaseMerge,
            DeleteBranchOnMerge: &full.DeleteBranchOnMerge,
        },

### [should-fix] `WithDefaults` and `Defined` hand-enumerate fields

- where: `pkg/config/org/org.go:129-161`
- concern: Adding a field to `RepoSettings` later will silently leave it undefaulted, and the tests would not catch it because they hardcode the same list. This PR fixes exactly this class of omission in `RepoRequest.Defined()` (`pkg/github/types.go:491`). An exhaustiveness test that reflects over `RepoSettings` and asserts every field is resolved would make future omissions fail loudly.
- excerpt: |
    func (r Repo) WithDefaults(defaults *RepoSettings) Repo {
        if defaults == nil {
            return r
        }
        if r.AllowSquashMerge == nil {
            r.AllowSquashMerge = defaults.AllowSquashMerge
        }
        ...
        return r
    }

### [nit] Embedding `RepoSettings` is a source break for downstream keyed literals

- where: `pkg/config/org/org.go:62-67`
- concern: YAML and JSON stay compatible, but Go callers outside the tree that build `org.Repo{HasIssues: ...}` with promoted fields stop compiling and need the nested `RepoSettings:` form. Worth a release-note mention if `pkg/config/org` importers matter.
- excerpt: |
    type Repo struct {
        RepoSettings
        Description *string                `json:"description,omitempty"`
        HomePage    *string                `json:"homepage,omitempty"`
        Visibility  *github.RepoVisibility `json:"visibility,omitempty"`

### [nit] `RepoDefaults` and `PruneRepoDefaults` are easy to confuse

- where: `pkg/config/org/org.go:166`
- concern: One is org-level inheritance for listed repos; the other prunes GitHub's built-in defaults from dumps. A short doc comment on `RepoDefaults` (only listed repos inherit, `null` means inherit, no per-repo opt-out) and a rename such as `PruneGitHubDefaults` would reduce future misreads.
- excerpt: |
    type Config struct {
        Metadata
        RepoDefaults *RepoSettings   `json:"repo_defaults,omitempty"`
        Teams        map[string]Team `json:"teams,omitempty"`

## Checked

- `nil` and empty `repo_defaults` are strict no-ops; `TestConfigureReposWithDefaults` covers both.
- Explicit `false` overrides a `true` default, explicit `true` overrides a `false` default, and `null`/omitted inherits — tested for create and update.
- Defaults are merged before both `CreateRepo` and the update delta, so creation and `--fix-repos` behave consistently.
- Unlisted repos are untouched and defaults alone manage nothing — explicitly tested.
- Repos declared `archived: true` are skipped; that path is covered.
- JSON and YAML marshal flat (no `RepoSettings`/`repo_settings` nesting) and round-trip through `UnmarshalStrict`; legacy `private` parsing still works.
- `PruneRepoDefaults` behavior is unchanged apart from the struct-literal rewrite.
- No new API endpoints, permissions, scopes, flags, or background work; the merge is local and cheap.
- `go test ./pkg/config/org ./pkg/github ./cmd/peribolos` passes locally, and CI on the head SHA is green (unit, race, lint, integration, image build).

## Open questions

- With defaults, an archived repo that is not declared `archived: true` can now get a delta and a rejected write. Should peribolos skip any repo GitHub reports as archived, or should the docs state that `archived: true` is required for archived repos?
- Should `--dump` emit `squash_merge_commit_title`/`squash_merge_commit_message` (with default pruning), or should the doc line say explicitly which settings dumps do not report?
- Should the upgrade note call out that configs which already set the squash commit fields will start applying them on the first run after upgrade, since those deltas were previously skipped?
- Are unknown keys inside `repo_defaults` intentionally ignored? Config loading is non-strict `yaml.Unmarshal` (`cmd/peribolos/main.go:200`), so a typo such as `visiblity:` silently disables a default across the whole org, and there is no local validation of the squash enum values.
