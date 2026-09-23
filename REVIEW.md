---
pr: kubernetes-sigs/prow#661
title: "Add support for git cherry-pick -x style commit messages"
head_sha: 05052e23ba428d40595917f28f2236f488b5b3a5
base: main
reviewed_at: 2026-09-23T12:48:17Z
verdict: request-changes
---

## Findings

### [blocking] SHA extraction rejects valid long-line patches
- where: `cmd/external-plugins/cherrypicker/server.go:850-860`
- concern: `bufio.Scanner` has a 64 KiB maximum token size by default. A valid GitHub mbox patch containing a long generated, minified, or lockfile line makes `extractOriginalSHAs` fail; with `--add-original-commit-id`, the handler then refuses to push an otherwise successful cherry-pick branch.
- excerpt: |
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        line := scanner.Text()
        if matches := fromPattern.FindStringSubmatch(line); matches != nil {
            shas = append(shas, matches[1])
        }
    }
    if err := scanner.Err(); err != nil {
        return nil, fmt.Errorf("error reading patch file: %w", err)
    }

## Checked

- The new option defaults to false, preserving existing cherrypicker behavior unless explicitly enabled.
- Rebase failures are aborted and reset to the pre-rebase HEAD before returning, and no branch is pushed after an extraction/rewrite failure.
- Commit messages are written through a temporary file rather than interpolated into the generated shell script.
- Unit coverage exercises empty input, invalid history, single- and multi-commit trailer ordering, and rebase rollback.

## Open questions

- Can the follow-up add a handler-level test for an enabled multi-commit mbox patch, plus regression coverage for a patch line larger than 64 KiB?
- Should the cherrypicker documentation describe `--add-original-commit-id`, its default-disabled rollout, and the changed commit hashes/messages when enabled?
