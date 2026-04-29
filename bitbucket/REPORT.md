# Automated AI Code Review — Executive Report (Bitbucket)

## What is this?

An automated code review system that uses AI to review every pull request before it reaches a human reviewer. It catches bugs, security vulnerabilities, and missing requirements — then posts its findings directly on the pull request for the developer to address.

This runs as a Bitbucket Pipeline step, fully integrated into the existing CI/CD workflow.

## Why are we doing this?

- **Human reviewers miss things.** Studies show that manual code review catches roughly 60% of defects. AI review acts as a second pair of eyes that checks every line.
- **Security issues slip through.** Developers focus on functionality. Security vulnerabilities like data leaks, injection attacks, or exposed credentials often go unnoticed until they become incidents.
- **Reviews create bottlenecks.** Developers wait hours or days for feedback. AI review provides initial feedback within minutes, so human reviewers can focus on architecture and design decisions rather than catching typos and bugs.

## What does it do?

When a developer submits code changes:

1. **Reads the PR description** — understands what the developer intended to build
2. **Reviews code quality** — looks for bugs, missing tests, logic errors, regressions
3. **Reviews security** — scans for common vulnerabilities (data leaks, injection attacks, authentication bypasses, exposed passwords)
4. **Checks completeness** — verifies that everything promised in the PR description was actually implemented
5. **Avoids noise** — reads existing review comments and does not repeat what was already said. Limits inline comments to prevent information overload.
6. **Posts results** — leaves specific comments on problem lines and a summary on the pull request

## What does the output look like?

Developers see two summary comments on every PR — one for code review, one for security:

- **Overall assessment** — a plain-language verdict on the quality and safety of the changes
- **Finding counts** — how many issues were found, broken down by severity
- **Changes walkthrough** — a quick-scan table showing what each file changed and why, so reviewers can orient themselves before diving into the code
- **Risk flags** — automatic flags like "this PR touches authentication code" or "no tests were added for these code changes" — highlights areas that need extra human attention
- **Suggested labels** — recommended tags for the PR to help with organization and routing

In addition, developers get **inline comments on specific lines** where issues were found, tagged by severity (critical, warning, suggestion, nitpick).

On subsequent pushes, old review comments are automatically replaced with fresh ones — no stale feedback cluttering the PR.

## What it does NOT do

- It does not approve or merge pull requests
- It does not replace human reviewers — it assists them
- It does not modify any code
- It does not have access to production systems, databases, or customer data
- It does not review draft pull requests

## Cost

- **Per review**: One AI session per pull request update, typically taking 2-10 minutes
- **Stale run detection**: If a developer pushes again before a review finishes, the older pipeline run detects it's stale and exits early — no wasted spend
- **Skip option**: Developers can skip reviews for trivial changes by adding `[skip-review]` to the PR title
- **Lighter reviews**: Documentation-only changes and lockfile updates get a faster, lighter review — no unnecessary analysis

## Security

- The AI has **read-only access** to the code diff and the ability to post comments
- It **cannot** push code, approve PRs, or access anything outside the repository
- The Claude CLI version is **pinned** to prevent unexpected behavior from upgrades
- Authentication tokens are stored as **encrypted pipeline variables** — not in the repository code
- The PR description is sandboxed to prevent it from being interpreted as instructions to the AI

## Platform

Bitbucket Cloud — runs as a custom Bitbucket Pipeline step. No official Anthropic Bitbucket integration exists, so this is a purpose-built implementation.

## Setup required

Two pipeline variables added to repository settings:

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Authenticates with the Claude Code API |
| `BB_TOKEN` | Authenticates with the Bitbucket API to post review comments |

No infrastructure, no servers, no maintenance. Runs entirely within Bitbucket Pipelines.

## Expected benefits

- Faster initial feedback on pull requests (minutes instead of hours)
- More consistent review coverage across all changes
- Earlier detection of security vulnerabilities before they reach production
- Human reviewers freed to focus on higher-value feedback (architecture, design, approach)
- Reduced risk of shipping bugs or security issues that were missed in review
- Risk flags help reviewers quickly identify which PRs need careful attention vs. a quick glance
- Changes walkthrough gives reviewers a fast orientation before they start reading code

## Limitations

- AI review quality depends on the clarity of the code and PR description
- It may occasionally flag non-issues (false positives) or miss subtle bugs (false negatives)
- It reviews individual pull requests in isolation — it does not understand multi-PR project context
- Comment deduplication is best-effort, not guaranteed
- Inline comments are capped at 10 per review to reduce noise — additional findings appear in the summary
