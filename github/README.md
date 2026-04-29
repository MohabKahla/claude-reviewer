# Claude Code Automated PR Review

AI-powered code review and security review that runs automatically on every pull request.

## How it works

When a pull request is opened or updated, Claude Code reviews the changes in two passes:

1. **Code Review** — checks for bugs, code quality issues, test gaps, and regressions
2. **Security Review** — checks for injection, auth bypass, secrets exposure, XSS, SSRF, and other vulnerabilities

Claude reads the PR description and verifies that everything promised is actually implemented. It also reads existing review comments to avoid posting duplicates.

Results are posted directly on the pull request as inline comments on specific lines and a summary comment with finding counts.

## Setup

### 1. Add the workflow file

Copy `claude-review-raw.yml` to your repository at `.github/workflows/claude-review.yml`.

### 2. Add the secret

Add `CLAUDE_CODE_OAUTH_TOKEN` to your repository secrets:

**Settings → Secrets and variables → Actions → New repository secret**

This token authenticates with the Claude Code API. Contact your team lead to get one.

### 3. Done

The review runs automatically on every eligible pull request. No other configuration needed.

## What gets reviewed

| PR type | Reviewed? |
|---------|-----------|
| Regular PR | Yes |
| Draft PR | No — skipped until marked ready |
| PR with `skip-review` in title | No |
| Fork PR | No — security measure |
| Dependabot PR | No |

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

## Review output

Each PR gets:

- **Inline comments** on specific lines where issues were found
- **Summary comment** with a table showing total findings and critical counts for both code and security reviews
- **Missing items section** listing anything promised in the PR description but not implemented

The summary comment updates in place on each new push — no comment pile-up.

## Skipping a review

Add `skip-review` anywhere in the PR title:

```
fix: update README skip-review
```

## Cost and performance

- One Claude Code session per PR update (~2-10 minutes depending on diff size)
- Cancels previous runs automatically if a new push arrives before the review finishes
- 20-minute timeout as safety limit

## Customizing

Edit the `prompt:` section in the workflow file to change review focus, add project-specific rules, or adjust the output format.

## Files

| File | Purpose |
|------|---------|
| `claude-review-raw.yml` | GitHub Actions workflow (copy to `.github/workflows/`) |
| `bitbucket-pipelines.yml` | Bitbucket Pipelines equivalent |
