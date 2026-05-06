# Claude Reviewer

AI-powered automated pull request review using [Claude Code](https://claude.com/claude-code). Drop the right config into your repo and Claude reviews every PR for bugs, regressions, missing tests, and security vulnerabilities — posting findings as inline comments and a summary on the PR.

## What it does

On every pull request open or update, Claude runs two review passes:

1. **Code review** — bugs, logic errors, missing test coverage, regressions, and whether the PR actually does what its description claims.
2. **Security review** — injection, auth bypass, exposed secrets, XSS, SSRF, path traversal, broken access control, and other common vulnerability classes.

Findings land on the PR as severity-tagged inline comments (Critical / Warning / Suggestion / Nitpick) plus a summary comment with a changes walkthrough, risk flags, and suggested labels.

Draft PRs, PRs marked `skip-review`, and PRs that only touch docs/lockfiles get skipped or lightly reviewed automatically.

## Pick your platform

This repo ships two independent integrations. Pick the one that matches your CI:

| Platform | Directory | Entry point |
|----------|-----------|-------------|
| GitHub Actions | [`github/`](./github) | [`github/README.md`](./github/README.md) |
| Bitbucket Pipelines | [`bitbucket/`](./bitbucket) | [`bitbucket/README.md`](./bitbucket/README.md) |

Each directory is self-contained — the platform README walks through setup, required secrets/tokens, and how to read the review output.

## Quick start

**GitHub** — copy [`github/claude-review-raw.yml`](./github/claude-review-raw.yml) to `.github/workflows/claude-review.yml` in your repo, add the `CLAUDE_CODE_OAUTH_TOKEN` secret, and you're done.

**Bitbucket** — copy [`bitbucket/bitbucket-pipelines.yml`](./bitbucket/bitbucket-pipelines.yml) to your repo root (or merge its `pull-requests` section into your existing pipelines file), then add `CLAUDE_CODE_OAUTH_TOKEN` and `BB_TOKEN` as secured pipeline variables.

In both cases you'll need a Claude Code OAuth token — contact your team lead for one.

## Repository layout

```
.
├── github/      GitHub Actions workflow + docs
├── bitbucket/   Bitbucket Pipelines config + docs
└── old/         Archived earlier iterations (not for use)
```

The `old/` directory is kept for reference only — use the `github/` or `bitbucket/` configs for any new setup.
