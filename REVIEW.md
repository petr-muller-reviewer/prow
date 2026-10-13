---
pr: kubernetes-sigs/prow#583
title: "`peribolos`:  Add Repository Fork Management Support"
head_sha: 4ec0010fc71a954f6e20a3f658bfd5d47c25307c
base: main
reviewed_at: 2026-10-05T20:16:37Z
verdict: request-changes
---

## Verdict

**Request changes.** Explicit `--fix-forks` removes the earlier silent opt-in path, and the author addressed several findings from the previous review. Four actionable defects remain for deployments that adopt fork management: malformed fork configuration can mutate an existing repository before failure; renaming and case mismatches can remove existing team access; and two config entries can reconcile the same actual repository. I found no confirmed runtime regression for unchanged configs without `fork:`.

## What this PR does

- Adds a `fork:` block to repository config and creates organization forks from an upstream repository through the GitHub API.
- Finds existing forks by their parent, waits for new forks, and passes their actual names to repository, collaborator, and team reconciliation.
- Makes fork creation and reconciliation explicitly opt-in with `--fix-forks`; `--fix-repos` no longer enables it.
- Records fork parents in `--dump` output and adds tests for creation, matching, validation, and access reconciliation.

Since previous review:

- The PR was rebased onto a newer `main` that includes organization-role management. The fork patch remains in six files and adds fixes for team repo mapping, `previously` renames, fork source parsing, validation order, and explicit opt-in.
- On 2026-10-02, `petr-muller` commented on five of these paths. `hoxhaeris` replied with code changes and reported live end-to-end checks.

## Findings

### [blocking] Stop downstream writes when fork source validation fails
- where: `cmd/peribolos/main.go:1648-1653,1239-1261`
- concern: With `--fix-forks --fix-repos`, an invalid `fork.from` returns an error and nil conflict information, but `configureOrg` defers that error and still runs repository and collaborator reconciliation. If an existing non-fork occupies the config key, its metadata or access can change before the run fails. This breaks existing behavior after opting into fork management; reject invalid fork config before dependent writes or exclude its entries from every downstream stage.
- excerpt: |
    if len(validationErrors) > 0 {
        sort.Slice(validationErrors, func(i, j int) bool {
            return validationErrors[i].Error() < validationErrors[j].Error()
        })
        return nil, nil, utilerrors.NewAggregate(validationErrors)
    }

### [blocking] Refresh the fork-name map after a `previously` rename
- where: `cmd/peribolos/main.go:1528-1553`
- concern: `configureForks` maps the config key to the fork's old name, and `configureRepos` now renames that fork when `previously` lists the old name. The map remains stale when collaborator and team reconciliation run. Team reconciliation can then grant the old path and remove the existing permission on the renamed repository, breaking access after feature adoption. Update the map to the new name after a successful rename, or pass the resolved post-rename name to later stages.
- excerpt: |
    if isFork && actualName != wantName {
        renameRequested := false
        for _, prev := range wantRepo.Previously {
            if strings.EqualFold(prev, actualName) {
                renameRequested = true
                break
            }
        }
        if !renameRequested {
            updateName = actualName
        }
    }

### [blocking] Compare conflicted repository names without regard to case
- where: `cmd/peribolos/main.go:2113-2121`
- concern: `configureForks` stores the config spelling in `conflictedForks`, while `ListTeamReposBySlug` returns GitHub's spelling. For a rejected fork configured as `My-Fork` but listed as `my-fork`, this exact-case check misses the conflict and schedules removal of the team's existing permission. Normalize both names before deciding whether access to a rejected repository may be changed.
- excerpt: |
    for haveRepo := range have {
        if conflictedForks.Has(haveRepo) {
            continue
        }
        if _, wantRepo := want[haveRepo]; !wantRepo {
            actions[haveRepo] = github.None
        }
    }

### [should-fix] Reject an adopted fork whose actual name has another config entry
- where: `cmd/peribolos/main.go:1754-1757`
- concern: If config key `foo` adopts an existing fork actually named `bar` while `bar` also has its own repository config entry, `validateRepos` accepts the distinct keys and both entries reconcile the same GitHub repository. Their metadata can overwrite each other in map iteration order. Reject the ambiguous mapping before repository or access reconciliation. This affects only feature adopters.
- excerpt: |
    if actualName, ok := forkParentIndex()[expectedUpstream]; ok {
        forkNames[repoName] = actualName
        repoLogger.WithField("actual_name", actualName).Info("fork of upstream already exists with different name")
        continue
    }

## Resolved

- At the prior reviewed head, team reconciliation did not map config keys to actual fork names. The new `configureTeamRepos` maps them; the `previously` rename path above still leaves a stale map.
- At the prior reviewed head, team reconciliation ignored fork conflicts. It now skips them for matching spellings; the case mismatch above remains.
- The earlier `previously` finding is addressed: `configureRepos` now requests the rename. The later access reconciliation needs the updated name, as described above.
- `fork.from` now uses `strings.Split` and rejects extra path segments.
- `validateRepos` now runs before fork creation, preventing a rejected repository-name collision from creating a fork first.

## Checked

- Reviewed the PR patch against its merge base at `f82256c4c4328dfc2ed2be653c64cf321765b04b`; the reviewed head is `4ec0010fc71a954f6e20a3f658bfd5d47c25307c`.
- Without `--fix-forks`, `configureForks` is skipped. With that flag and no `fork:` entries, it returns before a fork API call. `--dump` still adds `fork:` for existing forks, so its output changes even for deployments that have not enabled fork management.
- The fork client sends the target organization, requested name, and `default_branch_only`; its GitHub App token scope is the target organization. New forks are polled for availability, retrying only 404 responses.
- Tests for mapped team access, matching-spelling conflicts, `previously` rename, source parsing, and validation-before-creation cover those individual paths. They do not cover a full rename followed by access reconciliation, case-different conflict names, or two config entries resolving to one actual repository.
- `go test ./cmd/peribolos ./pkg/config/org ./pkg/github` passed at the reviewed head.

## Open questions

- Should an existing fork under a different name be adopted in place when `previously` does not authorize a rename, or should peribolos report a name mismatch? The PR description says the config key enforces the fork name.
- Are there out-of-tree implementations of the exported `github.RepositoryClient` interface? Adding `CreateForkInOrg` requires them to add the method before they compile.
