---
pr: kubernetes-sigs/prow#583
title: "`peribolos`:  Add Repository Fork Management Support"
head_sha: 54d1266b998cd4d92201c041031a3935e5dd36ce
base: main
reviewed_at: 2026-10-14T14:00:09Z
verdict: request-changes
---

# Review

## Verdict

**Request changes.** I found three paths where opting into fork management can change existing repository metadata or access after fork reconciliation fails or chooses an ambiguous fork. I found no confirmed regression for a deployment with no `fork:` config and `--fix-forks` disabled. That is the most important distinction for merge risk: the current findings affect adopters, including their existing behavior, rather than clearly endangering deployments that do not use the feature.

## What this PR does

- Adds an optional `fork:` block to repository config and an explicit `--fix-forks` flag; `--fix-repos` does not imply fork creation.
- Creates organization forks through the GitHub API, waits for new forks to appear, and recognizes existing forks by their parent.
- Passes a fork's actual GitHub name into repository, collaborator, and team permission reconciliation.
- Includes fork parents in `--dump` output and adds tests for fork creation, validation, renames, access mapping, and several error paths.

## Findings

### [blocking] Skip fork entries when the initial repository inventory fails
- where: `cmd/peribolos/main.go:1793-1797`
- concern: If this `GetRepos` call fails, `configureForks` returns no skip set, but `configureOrg` defers the error and continues into collaborator and team reconciliation. For example, with `--fix-forks --fix-collaborators` and a non-fork occupying the configured fork name, a transient inventory failure lets collaborator changes reach that unverified repository. Return all configured fork names in the skip set on this global failure, or stop dependent reconciliation. **Risk class: existing behavior can break for feature adopters.**
- excerpt: |
    repoList, err := client.GetRepos(orgName, false)
    if err != nil {
        return nil, nil, fmt.Errorf("failed to get repos: %w", err)
    }

### [blocking] Protect the physical fork name when rejecting duplicate aliases
- where: `cmd/peribolos/main.go:2296-2307`
- concern: Two config keys for the same upstream are correctly rejected and added to the skip set by config name. If the existing fork has a third, different name, the team want entries are skipped but this have loop does not recognize the physical name and removes its existing team permission. Track and protect the physical repo for rejected fork aliases, or avoid team repo removals when their target cannot be established. **Risk class: existing behavior can break for feature adopters.**
- excerpt: |
    for haveRepo := range have {
        if shouldSkipForkRepo(skipForks, haveRepo) {
            continue
        }
        if _, wantRepo := want[haveRepo]; !wantRepo {
            // should remove these permissions
            actions[haveRepo] = github.None
        }
    }

### [should-fix] Reject multiple existing forks of the same upstream
- where: `cmd/peribolos/main.go:1835-1837`
- concern: `CreateForkInOrg` explicitly allows another fork of an upstream under a different name. If two such forks already exist and the config key matches neither, this assignment overwrites the first match with the second; repository metadata and access are then reconciled against whichever fork appeared last in the API list. Detect the ambiguity and skip it instead of choosing a physical repository implicitly. **Risk class: existing behavior can break for feature adopters.**
- excerpt: |
    if full.Parent.FullName != "" {
        forkParentByUpstream[strings.ToLower(full.Parent.FullName)] = repo.Name
    }

## Resolved

- The prior invalid-`fork.from` finding is addressed: validation marks malformed entries as conflicted and the downstream stages skip them.
- The prior stale-name finding is addressed: a confirmed `previously` rename updates `forkNames` before collaborator and team reconciliation.
- The prior case-sensitive team-removal finding is addressed for repositories whose physical name matches the rejected config key apart from case.
- The prior duplicate-actual-repo finding is addressed: resolved names are checked for collisions and both config entries are skipped.

## Checked

- Compared PR head `54d1266b998cd4d92201c041031a3935e5dd36ce` with merge base `f82256c4c4328dfc2ed2be653c64cf321765b04b` and reviewed the fork code, shared repo/team paths, and relevant tests. The worktree's `HEAD` is a local review-artifact commit, so `head_sha` records the actual GitHub PR head.
- `--fix-forks` defaults to false and is independent of `--fix-repos`. With no `fork:` entries, `configureForks` returns before calling the GitHub API. Nil fork-name and skip maps leave the normal repository, collaborator, and team paths effectively unchanged; I found no confirmed non-adopter regression.
- `--dump` now records the parent of an existing fork even without `--fix-forks`, so dump output does change for organizations with forks. This is expected feature output; applying that config uses the `fork:` behavior described above.
- Target-organization token selection, the fork API request, creation wait, case-insensitive conflict skipping, and post-rename name propagation have direct tests.
- `go test ./cmd/peribolos ./pkg/config/org ./pkg/github` passed. The three finding scenarios lack targeted tests: inventory failure followed by access reconciliation, rejected aliases with a differently named physical fork, and multiple existing forks of one upstream.

## Open questions

- Are there out-of-tree implementations of the exported `github.RepositoryClient` interface? Adding `CreateForkInOrg` requires those implementations to add the method before they compile; I found no in-tree breakage.
