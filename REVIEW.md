---
pr: kubernetes-sigs/prow#583
title: "`peribolos`: Add Repository Fork Management Support"
head_sha: 627448bc90c0428d1cb4dea103ca81d8926c88b6
base: main
reviewed_at: 2026-09-23T19:22:12Z
verdict: request-changes
---

## Verdict

**Request changes.** A fork conflict can still lead to writes against the wrong repository, and a correctly sourced fork under a different name can lose team permissions. The other findings concern incomplete name and rename handling and malformed `fork.from` validation. All five require a `fork:` entry; unchanged existing configuration files do not exercise these paths.

## What this PR does

- Adds declarative `fork:` configuration and creates organization forks through GitHub's forks API.
- Waits for created forks, then reconciles their configured metadata and collaborators.
- Adds `--fix-forks`, inheriting from `--fix-repos` unless explicitly set.
- Emits a fork's parent in `--dump` output.

## Findings

### [blocking] Do not reconcile a repository after its fork validation fails
- where: `cmd/peribolos/main.go:1006-1027`
- concern: `configureForks` reports a same-name non-fork or wrong-upstream fork through `forkErr`, but `configureOrg` still runs repository and collaborator reconciliation with no successful mapping for that entry. Those paths treat the conflicting repository as desired and can mutate its metadata or collaborators despite the fork failure. Exclude failed fork entries from downstream reconciliation, or stop before mutating them.
- excerpt: |
    forkNames, forkErr = configureForks(client, orgName, orgConfig)
    if forkErr != nil {
        logrus.WithError(forkErr).Error("errors configuring some forks, continuing with partial results")
    }

### [blocking] Map fork names when reconciling team repository permissions
- where: `cmd/peribolos/main.go:1054-1056,1765-1797`
- concern: `configureForks` accepts a correctly sourced existing fork whose actual name differs from its config key, but `configureTeamRepos` receives no `forkNames` mapping. With `--fix-teams --fix-team-repos`, it attempts to grant permission on the nonexistent config-key name and can remove the team's existing permission on the real fork.
- excerpt: |
    for haveRepo := range have {
        if _, wantRepo := want[haveRepo]; !wantRepo {
            // should remove these permissions
            actions[haveRepo] = github.None
        }
    }

### [should-fix] Enforce the configured fork name
- where: `cmd/peribolos/main.go:1448-1457,1265-1269`
- concern: An existing fork of the requested upstream under a different name is treated as successfully reconciled. `configureRepos` then deliberately retains that actual name, so a successful run can leave the configured repository name absent; either rename the fork or report the mismatch as a conflict.
- excerpt: |
    if strings.ToLower(fullRepo.Parent.FullName) == expectedUpstream {
        forkNames[repoName] = repo.Name
        if strings.ToLower(repo.Name) == repoNameLower {
            repoLogger.Debug("fork already exists with correct upstream")
        } else {
            repoLogger.WithField("actual_name", repo.Name).Info("fork of upstream already exists with different name")
        }
        found = true
        break
    }

### [should-fix] Consider `previously` while locating existing forks
- where: `cmd/peribolos/main.go:1417-1424`
- concern: The fork search checks only the configured name and upstream repository name, ignoring `Repo.Previously`. If a fork exists under a previous name that differs from both, reconciliation sends a new fork request before normal repository rename handling, risking an API failure or a duplicate fork.
- excerpt: |
    candidates := []string{repoNameLower}
    if repoNameLower != upstreamRepoNameLower {
        candidates = append(candidates, upstreamRepoNameLower)
    }

### [should-fix] Reject `fork.from` values with extra path segments
- where: `cmd/peribolos/main.go:1363-1366`
- concern: `strings.SplitN(..., "/", 2)` accepts `owner/repo/extra`, then uses `repo/extra` as a GitHub API path segment. Require exactly two nonempty segments so invalid input fails validation, not as a malformed/404 request.
- excerpt: |
    parts := strings.SplitN(repoCfg.Fork.From, "/", 2)
    if len(parts) != 2 || parts[0] == "" || parts[1] == "" {
        validationErrors = append(validationErrors, fmt.Errorf("invalid fork from format %q for repo %s, expected 'owner/repo'", repoCfg.Fork.From, repoName))
    }

## Checked

- `CreateForkInOrg` sends target organization, requested fork name, and `default_branch_only`.
- The readiness loop retries only 404 responses and fails immediately for other GitHub errors.
- Case-insensitive matching is used for repository names and parent full names.
- `go test ./cmd/peribolos ./pkg/config/org ./pkg/github` passes. Focused tests cover normal creation, existing correct forks, wrong parents, asynchronous readiness, and basic invalid input; the cross-stage failure paths above lack integration coverage.
- With no `fork:` entries, `configureForks` returns before listing GitHub repositories. Existing configs without the new field do not trigger any of the five findings, even if the organization contains forks.
- `--dump` now emits `fork:` for existing forks, so adopting newly dumped config opts into fork management on the next `--fix-repos` run.

## Open questions

- Should `--fix-forks=false` suppress all reconciliation of `fork:` entries, not only their creation?
- Is source compatibility required for external Go consumers? Embedding `RepoMetadata` breaks keyed `org.Repo{Description: ...}` literals, and adding `CreateForkInOrg` to `github.RepositoryClient` breaks external implementations; serialized config keys remain flat.
