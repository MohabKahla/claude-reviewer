# Claude Code Automated PR Review — Bitbucket

AI-powered code review and security review that runs automatically on every pull request via Bitbucket Pipelines.

## How it works

When a pull request is opened or updated, Claude Code reviews the diff in two passes:

1. **Code Review** — checks for bugs, code quality issues, test gaps, and regressions
2. **Security Review** — checks for injection, auth bypass, secrets exposure, XSS, SSRF, and other vulnerabilities

Claude reads the PR description, verifies that everything promised is actually implemented, and reads existing review comments to avoid posting duplicates.

Results are posted directly on the pull request as inline comments on specific lines and a summary comment.

## Setup

### 1. Add the pipeline file

Copy `bitbucket-pipelines.yml` to the root of your repository (or merge the `pull-requests` section into your existing `bitbucket-pipelines.yml`).

### 2. Add pipeline variables

Go to **Repository settings → Pipelines → Repository variables** and add:

| Variable | Secured? | Description |
|----------|----------|-------------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Yes | Authenticates with Claude Code API. Contact your team lead to get one. |
| `BB_TOKEN` | Yes | Bitbucket API token (Bearer) for posting review comments. Create an App Password with `pullrequest:write` scope, or use a repository access token. |

### 3. Enable Pipelines

If Pipelines isn't already enabled: **Repository settings → Pipelines → Settings → Enable Pipelines**.

### 4. Done

The review runs automatically on every eligible pull request.

## How to use

### Reading the review output

Every PR gets a **code review summary** and a **security review summary** posted as comments. The code review summary contains:

| Section | What it tells you |
|---------|-------------------|
| **Summary** | Overall assessment + finding count |
| **Changes Walkthrough** | One-liner per changed file explaining what changed — scan this to orient yourself before reviewing |
| **Risk Flags** | Flags like `touches-auth`, `db-migration`, `no-tests-changed`, `api-surface`, `config-change` — tells you what areas need extra attention |
| **Suggested Labels** | Labels the reviewer thinks apply (e.g. `has-tests`, `security-sensitive`) |
| **Additional Findings** | Lower-priority findings that didn't fit in the inline comment budget |

You also get **inline comments** on specific changed lines. These are severity-tagged with icons:

| Icon | Severity | Meaning |
|------|----------|---------|
| :red_circle: | Critical | Must fix before merge |
| :yellow_circle: | Warning | Should fix, potential bug or risk |
| :blue_circle: | Suggestion | Improvement opportunity |
| :white_circle: | Nitpick | Minor style point |

### Skipping a review

Add `[skip-review]` in the PR title:

```
fix: update README [skip-review]
```

### Lighter reviews for docs and lockfiles

PRs that only change documentation files (.md), lockfiles, or generated files get a lighter review focused on correctness only — no style or architecture analysis.

### Draft PRs

Draft PRs are automatically skipped. The review runs when the PR is marked as ready.

### Stale runs

If a new commit is pushed while a review is in progress, the older pipeline detects the stale state and skips itself.

## What gets reviewed

| PR type | Reviewed? |
|---------|-----------|
| Regular PR | Yes |
| Draft PR | No — skipped until marked ready |
| PR with `[skip-review]` in title | No |
| Stale pipeline run (new push arrived) | No — automatically skips |

## What the review checks

### Code Review
- Bugs and logic errors
- Missing test coverage
- Regressions in existing behavior
- Whether PR description claims match actual implementation

### Security Review
- SQL/NoSQL injection
- Cross-site scripting (XSS)
- Authentication and authorization bypass
- Secrets or credentials in code
- Server-side request forgery (SSRF)
- Path traversal
- Insecure deserialization
- Broken access control

## Comment lifecycle

On each new push, the pipeline **deletes all previous bot comments** and posts fresh ones. This prevents stale feedback from lingering after the developer has addressed issues.

Inline comments are validated against the actual diff — only comments pointing to real changed lines are posted inline. Others are posted as general PR comments with a file reference.

## Cost and performance

- One Claude Code session per PR update (~2-10 minutes depending on diff size)
- 20-minute pipeline timeout as safety limit
- Claude CLI is cached between runs to speed up installation
- Comment budget limits inline comments to 10 per review to reduce noise — overflow goes to the summary

## Authentication

| Token | Purpose | How to get it |
|-------|---------|---------------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code API access | Contact your team lead |
| `BB_TOKEN` | Bitbucket API (post comments, read PR data) | Create an App Password with `pullrequest:write` scope |

Both must be stored as **secured pipeline variables** (not in the YAML file).

## Files

| File | Purpose |
|------|---------|
| `bitbucket-pipelines.yml` | Bitbucket Pipelines configuration (copy to repo root) |
