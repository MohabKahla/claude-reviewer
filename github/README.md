# Claude Code Automated PR Review

AI-powered code review and security review that runs automatically on every pull request.

## How it works

When a pull request is opened or updated, Claude Code reviews the changes in two passes:

1. **Code Review** — checks for bugs, code quality issues, test gaps, and regressions
2. **Security Review** — checks for injection, auth bypass, secrets exposure, XSS, SSRF, and other vulnerabilities

Claude reads the PR description, verifies that everything promised is actually implemented, and reads existing review comments to avoid posting duplicates.

Results are posted directly on the pull request as inline comments on specific lines and a summary comment.

## Setup

### 1. Add the workflow file

Copy `claude-review-raw.yml` to your repository at `.github/workflows/claude-review.yml`.

### 2. Add the secret

Add `CLAUDE_CODE_OAUTH_TOKEN` to your repository secrets:

**Settings → Secrets and variables → Actions → New repository secret**

This token authenticates with the Claude Code API. Contact your team lead to get one.

### 3. Done

The review runs automatically on every eligible pull request. No other configuration needed.

## How to use

### Reading the review output

Every PR gets a **summary comment** that updates in place (no comment pile-up). It contains:

| Section | What it tells you |
|---------|-------------------|
| **Code Review** | Overall assessment + finding/critical counts |
| **Security Review** | Security assessment + finding/critical counts |
| **Changes Walkthrough** | One-liner per changed file explaining what changed — scan this to orient yourself before reviewing |
| **Risk Flags** | Flags like `touches-auth`, `db-migration`, `no-tests-changed`, `api-surface`, `config-change` — tells you what areas need extra attention |
| **Missing from PR description** | Anything promised in the PR description but not implemented |
| **Suggested Labels** | Labels the reviewer thinks apply (e.g. `has-tests`, `security-sensitive`) — apply them manually if useful |
| **Additional Findings** | Lower-priority findings that didn't fit in the inline comment budget |

You also get **inline comments** on specific lines where issues were found. These are severity-tagged:

- Critical — must fix before merge
- Warning — should fix, potential bug or risk
- Suggestion — improvement opportunity
- Nitpick — minor style point

### One-click fixes

For simple mechanical fixes (typos, missing null checks, off-by-one errors), inline comments include a **GitHub suggestion block**. Click "Commit suggestion" to apply the fix directly from the PR page — no local checkout needed.

### Linked issue validation

If your PR description contains `fixes #123`, `closes #456`, or `resolves #789`, the reviewer fetches the linked issue and verifies your code actually addresses its requirements. Gaps are flagged in the summary.

### Skipping a review

Add `skip-review` anywhere in the PR title:

```
fix: update README skip-review
```

### Lighter reviews for docs and lockfiles

PRs that only change documentation files (.md), lockfiles, or generated files get a lighter review focused on correctness only — no style or architecture analysis.

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
- Whether linked GitHub issues are addressed

### Security Review
- SQL/NoSQL injection
- Cross-site scripting (XSS)
- Authentication and authorization bypass
- Secrets or credentials in code
- Server-side request forgery (SSRF)
- Path traversal
- Insecure deserialization
- Broken access control

## Cost and performance

- One Claude Code session per PR update (~2-10 minutes depending on diff size)
- Cancels previous runs automatically if a new push arrives before the review finishes
- 20-minute timeout as safety limit
- Comment budget limits inline comments to 10 per review to reduce noise — overflow goes to the summary

## Customizing

Edit the `prompt:` section in the workflow file to change review focus, add project-specific rules, or adjust the output format.

## Permissions

| Permission | Why |
|------------|-----|
| `contents: read` | Read repository code |
| `pull-requests: write` | Post review comments |
| `issues: read` | Fetch linked issue details for validation |

The reviewer cannot push code, approve PRs, or access anything outside the repository.

## Files

| File | Purpose |
|------|---------|
| `claude-review-raw.yml` | GitHub Actions workflow (copy to `.github/workflows/`) |
