---
pr: kubernetes-sigs/prow#583
title: "`peribolos`: Add Repository Fork Management Support"
head_sha: 627448bc90c0428d1cb4dea103ca81d8926c88b6
base: main
reviewed_at: 2026-09-10T12:53:16Z
verdict: request-changes
---

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
    } else if err := configureRepos(opt, client, orgName, orgConfig, forkNames); err != nil {

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
- Focused tests cover normal creation, existing correct forks, wrong parents, asynchronous readiness, and basic invalid input; neither finding has integration coverage.

## Open questions

- Should `--fix-forks=false` suppress all reconciliation of `fork:` entries, not only their creation?

