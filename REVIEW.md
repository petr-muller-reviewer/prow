---
pr: kubernetes-sigs/prow#1034
title: "add dry-run flag to bumper and fix an error in the bumper"
head_sha: 7b32f3d6fc56b9b877491b9ff7df65ab4411dba8
base: main
reviewed_at: 2026-10-27T20:17:29Z
verdict: approve
---

# Review

## Verdict

Correct fix for the actual CI breakage, plus a dry-run mode that is cleanly kept away from the commit/push/PR code; nothing here blocks. One wiring trap is worth following up — `dryRun: true` in the YAML config is silently overwritten by the flag default and would run the real bump — and the dry-run happy path deserves a test.

## What this PR does

- Adds `bumper.Options.DryRun` and the `--dry-run` flag; `Run` short-circuits into `processDryRun`, which applies the change funcs, then logs the would-be PR title/body and leaves the tree uncommitted.
- Relaxes `validateOptions` for dry runs: no GitHub App credentials, no fork token/remote name, no Gerrit cookiefile (Gerrit author/identifier/host stay required).
- Fixes the broken `ci-prow-autobump-base-images` job: `includedConfigPaths` is no longer mandatory when `extraFiles` is set, which is exactly the shape of `hack/autobump.yaml`.
- Rewrites the `includedConfigPaths` doc comment and adds validation tests for both the dry-run relaxations and the extraFiles-only case.
- Adds `TestRunDryRun`, covering error propagation from a change func.

## Findings

### [should-fix] `dryRun: true` in the config is silently discarded
- where: `cmd/generic-autobumper/main.go:206-217`
- concern: `parseOptions` unmarshals the config into `bumper.Options` (line 206), and the new field carries `json:"dryRun"` (`bumper.go:69`); the docs send config authors to `bumper.Options`. But line 217 unconditionally overwrites it with the flag value, whose default is `false`. Anyone who sets `dryRun: true` in YAML gets a real bump — commit, push, PR — instead of a preview. Either keep the config value when the flag was not passed (mirroring the `labelsOverride != nil` guard at line 210) or make the option clearly CLI-only. `SkipPullRequest` and `Signoff` already have the same overwrite pattern, but for a safety switch the silent inversion matters more.
- excerpt: |
    	if labelsOverride != nil {
    		pro.Labels = labelsOverride
    	}
    	if o.OncallGroup == "" {
    		o.OncallGroup = defaultOncallGroup
    	}
    	pro.SkipPullRequest = skipPullRequest
    	pro.DryRun = dryRun
    	pro.Signoff = signoff

### [nit] Dry-run happy path is untested
- where: `cmd/generic-autobumper/bumper/bumper_test.go:215-231`
- concern: The only new `Run` test asserts that a change-func error is propagated. Nothing covers the advertised behavior — changes applied, `PRTitleBody` called, no commit produced. A `t.TempDir()` + `git init` fixture with a change func that writes a file would pin that down cheaply; right now the "does not commit" guarantee is only enforced by reading the code.
- excerpt: |
    	prh := &fakePRHandler{changeErr: sentinel}
    	o := &Options{
    		GitHubOrg:  "whatever-org",
    		GitHubRepo: "whatever-repo",
    		DryRun:     true,
    	}

    	err := Run(context.Background(), o, prh)
    	if !errors.Is(err, sentinel) {
    		t.Fatalf("Expected Run to return the change func error, got %v", err)
    	}

### [nit] A dirty checkout makes the dry-run preview lie
- where: `cmd/generic-autobumper/bumper/bumper.go:258-269`
- concern: `HasChanges()` is `git status --porcelain` over the whole repo, so a pre-existing dirty tree makes `changed` true even when the bump produced nothing, and the logged body (built from an empty `images` map) says "No <prefix> changes." while the title says the images would be updated. The normal path has the same weakness, but here it is the user-facing output of a preview mode. Consider recording/requiring a clean tree before the change funcs run, so a local dry run reports only the bump's own changes.
- excerpt: |
    	changed, err := HasChanges()
    	if err != nil {
    		return fmt.Errorf("checking changes: %w", err)
    	}
    	if !changed {
    		logrus.Info("[Dryrun] Nothing changed from all functions.")
    		return nil
    	}

    	summary, body := prh.PRTitleBody()
    	logrus.Infof("[Dryrun] Changes left uncommitted in the working tree; skipping commit, push and PR creation.\nPR title: %s\nPR body:\n%s", summary, body)

### [question] `skip-if-no-oncall` short-circuits a dry run
- where: `cmd/generic-autobumper/main.go:627-633`
- concern: `main` returns before `bumper.Run` when `SkipIfNoOncall` is set and nobody is oncall, so `--dry-run` produces no diff at all in that state. The field comment says read-only lookups still happen, which suggests the lookup is intended, but is exiting early the intended preview behavior? If not, gate this check on `!pro.DryRun`.
- excerpt: |
    	if o.SkipIfNoOncall {
    		if !isOncallActive(o.OncallAddress, o.OncallGroup) {

    			logrus.Info("`skip-if-no-oncall` is configured and there is no active oncall. Skip bumping.")
    			return
    		}
    	}

### [question] Dry run still requires fields it never uses
- where: `cmd/generic-autobumper/bumper/bumper.go:197-202`
- concern: `processDryRun` uses neither org/repo nor git name/email, but they remain mandatory (and `bumper_test.go:109` enshrines the org requirement). `hack/autobump.yaml` supplies them, so this mostly affects ad-hoc configs. Is requiring the full production config shape deliberate, so `--dry-run` also validates what would run for real? If not, those checks could be skipped when `DryRun` is set.
- excerpt: |
    	if (o.GitEmail == "") != (o.GitName == "") {
    		return fmt.Errorf("gitName and gitEmail must be specified together")
    	}
    	if o.GitHubOrg == "" || o.GitHubRepo == "" {
    		return fmt.Errorf("gitHubOrg and gitHubRepo are mandatory when skipPullRequest is false or unspecified")
    	}

## Checked

- Confirmed the reported failure is real: the linked build log ends with `error="includedConfigPaths is mandatory"`, and `hack/autobump.yaml` (used by `ci-prow-autobump-base-images`) specifies only `extraFiles: [".ko.yaml"]`. `updateReferences` loops over `IncludedConfigPaths` and then `ExtraFiles`, so an empty include list is a no-op — the fix is correct and sufficient.
- `go test ./cmd/generic-autobumper/...` passes (all four packages).
- The dry-run branch runs before any GitHub client construction / `secret.Add`, so no token or App credentials are needed; `PRTitleBody`'s only external call is the read-only oncall HTTP GET.
- Gerrit mode: dry run skips the cookiefile requirement and never reaches `processGerrit`, so no git config or remote mutation happens.
- No other in-repo callers of `bumper.Run`/`bumper.Options` that would need the flag wired through.

## Open questions

- Would it help for `--dry-run` to also log the per-change commit messages (or the resulting `git diff`), so the preview is self-contained instead of requiring a separate `git diff`?

## Followups

Post-merge followups for PR #1034 (merged as `860d730d7225d4c4203ed2e3b6e1faa906ca4aca`). Six candidates walked, six accepted: two probes, four deliverables. The one review round (cblecker) was fully addressed; the `dryRun` config-clobber finding above was an explicit review decision (flags authoritative), so it is recorded here as a tag/precedence cleanup rather than re-litigated.

### tests: dry-run happy path is untested

- Mode: probe
- Category: tests
- Necessity: should
- Where: `cmd/generic-autobumper/bumper/bumper_test.go:215` (unfixed from the review above)
- Why: nothing pins "changes applied, `PRTitleBody` called, nothing committed" — the property the flag exists for.
- Prompt:

```text
In kubernetes-sigs/prow, following PR #1034 ("add dry-run flag to bumper and fix an error in the bumper", merged as 860d730d7225d4c4203ed2e3b6e1faa906ca4aca), write a throwaway probe test for the generic-autobumper dry-run path. The implementation is `processDryRun` in `cmd/generic-autobumper/bumper/bumper.go`, dispatched from `Run` when `Options.DryRun` is true; add the probe as uncommitted code in `cmd/generic-autobumper/bumper/bumper_test.go`.

Exercise these behaviours against a temporary git repository (`t.TempDir()` + `git init`, chdir in the test and restore cwd afterwards):

1. A change func that writes a file, with `DryRun: true`: after `Run`, the file is modified, `PRTitleBody` is called exactly once, and HEAD is unchanged (no commit created) — compare `git rev-parse HEAD` and `git log --oneline` counts before/after.
2. A change func that returns a sentinel error: `Run` returns an error wrapping it, `PRTitleBody` is not called, and no commit exists.
3. A change func that changes nothing: `Run` returns nil and `PRTitleBody` is not called.
4. A repository that is already dirty before `Run` (one modified or untracked file) plus a change func that changes nothing: observe whether `Run` logs a would-be PR body even though the bump itself changed nothing.

Triage every failure: classify it as a real implementation bug or a wrong test expectation, with evidence (actual vs expected). If you confirm a real bug, apply the minimal fix in the implementation and nothing else. If behaviour 4 reproduces, report it but do NOT fix it here — a separate followup owns that fix. If no real bug is found, delete the probe tests and change no files.

Acceptance criteria: every behaviour above is exercised; each failure is classified with evidence; any confirmed bug gets a minimal implementation fix; if no real bug is found, `git status` shows no changed files. Report which behaviours passed and the observed outcome of behaviour 4 either way.

Scope guard: this is a probe — do not commit the tests, delete them before finishing, and do not fix test-only problems, style, or unrelated code. Do not touch the non-dry-run paths. Do not add new public API. Do not fix the dirty-tree preview issue.
```

### tests: nothing validates the real autobump config

- Mode: probe
- Category: tests
- Necessity: should
- Where: `hack/autobump.yaml`; `cmd/generic-autobumper/main_test.go`
- Why: the exact breakage this PR fixed (validator drift vs the config the periodic job runs) went unnoticed because no test ever loads `hack/autobump.yaml`.
- Prompt:

```text
In kubernetes-sigs/prow, following PR #1034 (merged as 860d730d7225d4c4203ed2e3b6e1faa906ca4aca), probe whether the production autobump config still validates. The config used by the periodic job `ci-prow-autobump-base-images` is `hack/autobump.yaml`; the validator that broke was `validateOptions` in `cmd/generic-autobumper/main.go`. Nothing in the repo loads that file today.

Write uncommitted probe code in `cmd/generic-autobumper/main_test.go` that:

1. Reads `../../hack/autobump.yaml` and unmarshals it into `options` and into `bumper.Options` (sigs.k8s.io/yaml).
2. Asserts `validateOptions(&opts)` returns nil.
3. Exercises bumper-level validation without network: copy the resulting `bumper.Options`, force `DryRun = true`, and call `bumper.Run(context.Background(), &opts, fakeHandler)` from a temporary git repo with a fake `PRHandler` that returns no change funcs, asserting no validation error. If that is not reachable without mutating the repo, say so instead of forcing it.
4. Asserts the parsed config still has the expected shape: `ExtraFiles` contains `.ko.yaml`, `TargetVersion` is `latest`, and at least one prefix is present.

Run the probe on the current tree. Triage each failure as a real config/validator mismatch or a wrong expectation, with evidence. If a real mismatch exists, apply the minimal fix in the validator or config and nothing else; if everything passes, delete the probe and change no files.

Acceptance criteria: every check above was run; each failure is classified with evidence; a minimal fix only for a confirmed mismatch; otherwise no changed files. Report the commands used and their results.

Scope guard: this is a probe — tests are throwaway and must be deleted before finishing. Do not add a permanent test in this run; do not change the periodic job definition; do not touch unrelated validators or config files.
```

### docs: `--dry-run` and `--skip-pullrequest` are undocumented

- Mode: deliver
- Category: docs
- Necessity: should
- Where: `site/content/en/docs/components/cli-tools/generic-autobumper.md`; `cmd/generic-autobumper/main.go:189-190`
- Why: the new preview mode is undiscoverable, and the older `--skip-pullrequest` flag still has an empty help string despite different semantics.
- Prompt:

```text
In kubernetes-sigs/prow, following PR #1034 (merged as 860d730d7225d4c4203ed2e3b6e1faa906ca4aca), document the new generic-autobumper `--dry-run` mode. Relevant files: `site/content/en/docs/components/cli-tools/generic-autobumper.md` and the flag definitions in `cmd/generic-autobumper/main.go` (around lines 189-190).

Do:

1. Add a short "Preview changes locally" section to the docs page: `--dry-run` applies the bump to the working tree, logs the PR title/body it would use, and does not commit, push, or create a PR/CR; it needs no GitHub token or App credentials (read-only upstream-version and oncall lookups still happen); the working tree is left dirty so `git diff` shows the change; it is a CLI flag, not a YAML config key.
2. In the same section, distinguish it from `--skip-pullrequest`: that one commits locally but skips push and PR creation, and still needs the token/org/repo/remote settings; `--dry-run` does not commit at all.
3. Give `--skip-pullrequest` a real help string in `cmd/generic-autobumper/main.go` (currently `""`), e.g. "Commit the changes locally but do not push the branch or create/update the PR."
4. Note in the docs that CLI flags take precedence over YAML config values.

Acceptance criteria: the page is valid Hugo Markdown (headings/links intact), the help string is non-empty, every statement matches the code in `bumper.go`/`main.go` (verify each claim against the source before writing it), and `go build ./cmd/generic-autobumper/...` passes.

Scope guard: docs plus the one flag help string only; do not change behaviour, do not document `dryRun`/`skipPullRequest` as working config keys, and do not touch unrelated docs.
```

### bugfix: a dirty checkout makes the dry-run preview lie

- Mode: deliver
- Category: bugfix
- Necessity: could
- Where: `cmd/generic-autobumper/bumper/bumper.go:258-269`
- Why: repo-wide `HasChanges()` treats pre-existing local edits as the bump's changes, so the logged title/body can describe a bump that would not happen.
- Prompt:

```text
In kubernetes-sigs/prow, following PR #1034 (merged as 860d730d7225d4c4203ed2e3b6e1faa906ca4aca), fix the dry-run preview's dependence on pre-existing working-tree dirt.

Context: `processDryRun` in `cmd/generic-autobumper/bumper/bumper.go` runs the change funcs and then calls `HasChanges()` (repo-wide `git status --porcelain`). If the checkout was already dirty before the run, `changed` is true even when the bump changed nothing, and the code logs a would-be PR title/body built from empty replacement maps ("Update ... images as necessary" / "No ... changes.").

Do: snapshot `git status --porcelain` before running the change funcs and make dry-run report only changes it caused. Prefer failing fast with a clear error when the tree is not clean at the start (e.g. "refusing to dry-run on a dirty working tree; commit or stash first"), since that is unambiguous. If you find a concrete legitimate use that fail-fast breaks, instead subtract the pre-existing changed paths from the post-run status and treat only new paths as changes; say why you chose that route. Update the `--dry-run` help text / `DryRun` doc comment if the chosen behaviour is user-visible.

Acceptance criteria: a unit test in `cmd/generic-autobumper/bumper/bumper_test.go` covering (a) dirty tree + change func that changes nothing → `Run` does not call `PRTitleBody` (fail-fast error or "nothing changed", per your choice), and (b) clean tree + change func that writes a file → `PRTitleBody` called once and no commit created; `go test ./cmd/generic-autobumper/...` passes; the non-dry-run paths are untouched.

Scope guard: only `processDryRun` plus its tests/help text; do not change `HasChanges`, the GitHub/Gerrit paths, or PR-title generation; pick one approach, do not add both.
```

### tech-debt: four copies of the change-apply loop

- Mode: deliver
- Category: tech-debt
- Necessity: could
- Where: `cmd/generic-autobumper/bumper/bumper.go:252-256, 302-320, 376-392, 464-496`
- Why: the dry-run path added a fourth copy of the same loop; future changes to change detection/logging will only land in one copy.
- Prompt:

```text
In kubernetes-sigs/prow, following PR #1034 (merged as 860d730d7225d4c4203ed2e3b6e1faa906ca4aca), de-duplicate the change-apply loop that now exists four times in `cmd/generic-autobumper/bumper/bumper.go`: `processDryRun` (~252), `processGitHub` (~302), `processGitHubAppAuth` (~376), and `processGerrit` (~464).

Do: extract a shared helper, e.g. `applyChanges(ctx context.Context, prh PRHandler, onChanged func(i int, msg string) error) (anyChange bool, err error)`, that iterates `prh.Changes()`, calls each func, checks `HasChanges()`, logs the "Nothing changed, skip ..." message, and invokes `onChanged(i, msg)` when something changed. The callback carries the mode-specific work (commit, or a no-op for dry-run). Use it from all four call sites and preserve current behaviour exactly; the three real paths already check `HasChanges` per function, so keep that ordering. If you unify dry-run onto per-function checks, verify the resulting log output is still sensible and that the final "nothing changed" / PR-body behaviour is unchanged.

Acceptance criteria: `go test ./cmd/generic-autobumper/...` passes; no behavioural change (same commands, same order, same errors, same decisions); the diff only moves the loop logic and updates call sites.

Scope guard: behaviour-preserving refactor only — no new features, no validation changes, no dry-run semantic changes, no unrelated functions.
```

### tech-debt: config keys the flag defaults always clobber

- Mode: deliver
- Category: tech-debt
- Necessity: could
- Where: `cmd/generic-autobumper/bumper/bumper.go:61-69`; `cmd/generic-autobumper/main.go:206-218`; docs config example
- Why: the review decided flags are authoritative, but `dryRun`/`skipPullRequest`/`signoff` still carry JSON tags while the docs point config authors at `bumper.Options` — the schema advertises keys that are silently ignored.
- Prompt:

```text
In kubernetes-sigs/prow, following PR #1034 (merged as 860d730d7225d4c4203ed2e3b6e1faa906ca4aca), encode the reviewed precedence rule for option fields that exist both as YAML config keys and CLI flags.

Context: `parseOptions` in `cmd/generic-autobumper/main.go` unmarshals the config into `bumper.Options` (line ~206) and then unconditionally assigns `pro.SkipPullRequest`, `pro.DryRun`, and `pro.Signoff` from flag variables (lines ~216-218). During PR #1034 review, cblecker asked for exactly this (flags authoritative) because a stray `dryRun: true` in a job config would otherwise turn the periodic bump into a silent no-op. But the three fields still carry `json:"dryRun"`, `json:"skipPullRequest"`, and `json:"signoff"` tags, and `site/content/en/docs/components/cli-tools/generic-autobumper.md` points config authors at `bumper.Options` and shows `skipPullRequest: false` in its example — so the config surface advertises keys that are silently ignored.

Do: make the precedence explicit and consistent. Preferred: mark the three fields `json:"-"` (as `GitHubOptions` already is) so they cannot be set from YAML, and remove or annotate `skipPullRequest` in the docs example as flag-only. Keep the flag assignments as-is. If you instead believe config should be honoured when the flag is not explicitly passed, implement that with `flag.Visit`, document the precedence, and make sure a config `dryRun: true` cannot silently suppress a periodic job (log loudly at startup); say why you chose it.

Acceptance criteria: `yaml.Unmarshal([]byte("dryRun: true\nskipPullRequest: true\nsignoff: true\n"), &bumper.Options{})` leaves all three false — add this as a unit test in `cmd/generic-autobumper/bumper/bumper_test.go`; the docs example and the Options Go doc comments no longer imply these are usable config keys; `go test ./cmd/generic-autobumper/...` passes.

Scope guard: only these three fields, their docs, and one small test; do not change flag parsing, do not add config support for other fields, and do not touch other `Options` fields.
```
