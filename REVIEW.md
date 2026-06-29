---
pr: kubernetes-sigs/prow#730
title: "Add gemini-agent Prow plugin"
head_sha: 27bec9f35515e125802bdbd1fa5e6c5bc0f9e551
base: main
reviewed_at: 2026-06-03T12:17:07Z
verdict: approve-with-suggestions
---

## Findings

### [should-fix] errors.As with value type may silently fail against real SDK
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:125-129`
- concern: `var apiErr genai.APIError` (value type) as `errors.As` target. Works if SDK returns values with value `Error()` receiver. If SDK returns `*genai.APIError`, match silently fails and rate-limit retries stop working. Tests pass because fakes construct value types directly. Use `var apiErr *genai.APIError` instead.
- excerpt: |
    var apiErr genai.APIError
    if errors.As(err, &apiErr) {
        return apiErr.Code == http.StatusTooManyRequests
    }

### [should-fix] Empty response conflated with safety filter block
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:254-265`
- concern: When `resp.Text()` returns empty, error says "response was blocked by content safety filters." But `Text()` could be empty for other reasons (empty completion, function-call-only response). Check `resp.Candidates` and `FinishReason` to distinguish.
- excerpt: |
    text := strings.TrimSpace(resp.Text())
    if text == "" {
        return "", errors.New("response was blocked by content safety filters")
    }

### [should-fix] GeminiAgentConfig in shared plugins.Configuration struct
- where: `pkg/plugins/config.go:66-67`
- concern: No other external plugin puts its config in the shared `plugins.Configuration` struct. Cherrypicker and needs-rebase manage their own config independently. This creates a precedent that would bloat the shared struct if followed. Plugin should own its own config file or use flags. Flagged by Maintainability reviewer as the most significant concern.
- excerpt: |
    GeminiAgents []GeminiAgentConfig `json:"gemini_agents,omitempty"`

### [should-fix] Unbounded goroutine fire-and-forget with no shutdown drain
- where: `cmd/external-plugins/geminiagent/server.go:70-74`
- concern: `go func()` dispatches work with no WaitGroup, semaphore, or drain during shutdown. A 10-minute Gemini request could be silently killed when `interrupts.WaitForGracefulShutdown()` returns. Also logged at `Info` level — should be `Error` for production visibility. Flagged by both Maintainability and Deployment Risk reviewers.
- excerpt: |
    go func() {
        if err := s.handleIssueComment(l, ice); err != nil {
            l.WithError(err).Info("Gemini agent failed.")
        }
    }()

### [should-fix] Duplicate import alias in main.go
- where: `cmd/external-plugins/geminiagent/main.go:31-32`
- concern: `flagutil` and `prowflagutil` both import `sigs.k8s.io/prow/pkg/flagutil`. The unaliased import is used only for `flagutil.OptionGroup`. Drop the unaliased import and use `prowflagutil.OptionGroup`. Note: matches needs-rebase pattern, but worth cleaning up.
- excerpt: |
    "sigs.k8s.io/prow/pkg/flagutil"
    prowflagutil "sigs.k8s.io/prow/pkg/flagutil"

### [should-fix] Credential discovery error silently swallowed
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:233-252`
- concern: When `projectID` is empty and `google.FindDefaultCredentials` fails, the error is discarded and `projectID` stays empty. The genai client creation proceeds with an empty project, failing later with a less informative error. At minimum log the error.
- excerpt: |
    if projectID == "" {
        creds, err := google.FindDefaultCredentials(ctx, cloudPlatformScope)
        if err == nil {
            projectID = creds.ProjectID
        }
    }

### [should-fix] Environment variables as primary configuration
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:81,234-235`
- concern: Six env vars (`AI_AGENT_PROJECT_ID`, `AI_AGENT_MODEL`, `AI_AGENT_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, `GCLOUD_PROJECT`) read at call time, not startup. Not discoverable via `--help`, not loggable at startup, not documented. Diverges from Prow convention of flags and config files. Flagged by both Maintainability and Deployment Risk reviewers.

### [should-fix] Disconnected context ignores graceful shutdown
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:297`
- concern: `context.WithTimeout(context.Background(), requestTimeout)` creates a context disconnected from any parent. In-flight Gemini requests won't be cancelled during graceful shutdown — could block for up to 10 minutes.
- excerpt: |
    ctx, cancel := context.WithTimeout(context.Background(), requestTimeout)

### [should-fix] Client re-created per invocation with credential discovery I/O
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:372-390`
- concern: `newVertexGeminiClient` runs `google.FindDefaultCredentials` (potential file I/O) on every `/gemini-agent` comment. Cache the client or project ID with `sync.Once`.

### [should-fix] Missing test coverage for team-based authorization
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent_test.go:725-727`
- concern: `TestIsAllowed` never tests the positive `AllowedTeams` path. Fake `TeamBySlugHasMember` always returns false.
- excerpt: |
    func (f *fakeGitHubClient) TeamBySlugHasMember(_, _ string, _ string) (bool, error) {
        return false, nil
    }

### [should-fix] Prompt injection risk from user-controlled content
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:522-523`
- concern: Raw issue bodies, PR bodies, and comments passed directly into Gemini prompt. `isAllowed` gates who triggers the plugin, but content comes from anyone who can write on the issue. Should at minimum be documented.

### [nit] rand.Int64N panic risk if backoff is zero
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:319`
- concern: Current constants prevent zero backoff, but no guard. `rand.Int64N(0)` panics. One-line `if backoff <= 0` check would make it robust.
- excerpt: |
    jitter := time.Duration(rand.Int64N(int64(backoff) / 4))

### [nit] Magic fractions for truncation limits
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:522-523`
- concern: `maxPatchBytes/4` and `maxPatchBytes/8` not self-documenting. Named constants would clarify intent.

### [nit] Per-request os.Getenv for model
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:220`
- concern: `resolveConfig` reads `os.Getenv("AI_AGENT_MODEL")` per-request. Behavior can change mid-process without logging.

### [nit] No config validation for GeminiAgentConfig
- where: `pkg/plugins/config.go:105-116`
- concern: No `Validate()` call from `Configuration.Validate()`. Other plugin configs with `Repos` fields have validation.

### [nit] genai dependency in top-level go.mod
- where: `go.mod`
- concern: `google.golang.org/genai v1.57.0` added to top-level go.mod. All Prow builds pull it even if this plugin is not deployed. A separate Go module for the plugin would isolate this. Flagged by Deployment Risk reviewer.

### [question] Only handles issue_comment events
- where: `cmd/external-plugins/geminiagent/server.go:63-77`
- concern: Plugin only handles `issue_comment` events. `/gemini-agent` from a PR review comment silently ignored — no error, no log. Is this intentional? Flagged by Maintainability reviewer.

### [question] Default model choice
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:188`
- concern: Default model is `gemma-4-26b-a4b-it` (Gemma). Intentional over a first-party Gemini model?
- excerpt: |
    defaultModel = "gemma-4-26b-a4b-it"

### [question] AI-generated response attribution
- where: `cmd/external-plugins/geminiagent/plugin/gemini-agent.go:477`
- concern: Responses posted as regular GitHub comments with no indication they are AI-generated. Should there be an attribution line?

## Checked
- External plugin architecture: crash-isolated from hook, genai dependency not in hook binary
- `main.go` follows established Prow external plugin patterns (compared needs-rebase, cherrypicker)
- Webhook HMAC validation via `github.ValidateWebhook`
- `dry-run` defaults to `true` (safe default)
- Health checks and graceful shutdown via `pjutil.NewHealthOnPort` and `interrupts.ListenAndServe`
- `GitHubClient` interface correctly exported, well-scoped to methods actually used
- `genericCommentHandler` function type enables clean DI for testing
- `server_test.go` covers event generalization, config forwarding, nil handler guard
- `var _ plugin.GitHubClient = (*fakeGitHubClient)(nil)` interface compliance check
- Authorization flow (org member, collaborator, team checks) is correct and fail-closed
- Rate limiting: token bucket + exponential backoff with jitter
- Context collection handles both issue and PR paths, respects byte budgets
- Safety settings (BLOCK_MEDIUM_AND_ABOVE) explicitly configured
- Response truncation (60KB) prevents oversized comments
- `markdown.DropCodeBlock` prevents command parsing inside code blocks
- UTF-8 boundary handling in `truncate` is correct
- Config placement near `ExternalPlugins` in `config.go` (location is appropriate even if shared-struct precedent is questionable)
- `plugin-config-documented.yaml` updated with new config block
- `HelpProvider` signature matches external plugin pattern
- Deployment risk is LOW: purely additive, explicit opt-in, all config fields optional with omitempty
- No breaking changes to existing configurations or behavior
- Safe rollback: removing deployment and config returns to prior state

## Open questions
- Is `gemma-4-26b-a4b-it` the intended default, or should it be a Gemini model?
- Should `pull_request_review_comment` events also be handled, or is `issue_comment`-only intentional?
- Should the plugin's response include an attribution line to make it clear the comment is AI-generated?
- Is the 10-minute request timeout appropriate given it blocks graceful shutdown?
- Should `GeminiAgentConfig` move out of the shared `plugins.Configuration` to match how other external plugins handle config?
