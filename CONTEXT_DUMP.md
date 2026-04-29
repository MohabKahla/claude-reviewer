# CI/CD AI Review — Full Context Dump

**Generated**: 2026-04-27  
**Purpose**: Carry-over context for continuing work in another directory

---

## 1. Current Architecture

### GitHub Actions (`.github/workflows/claude-review.yml`)

**Trigger**: `pull_request` on `opened`, `synchronize`, `ready_for_review`, `reopened`

**Skip conditions** (in `if:` block):
- `draft == false`
- `!contains(title, '[skip-review]')`
- Same-repo only (`head.repo.full_name == github.repository`) — blocks forks
- `github.actor != 'dependabot[bot]'`

**Concurrency**: `claude-review-${{ PR_NUMBER }}` with `cancel-in-progress: true`

**Permissions**: `contents: read`, `pull-requests: write`

**Auth**:
- `CLAUDE_CODE_OAUTH_TOKEN` from secrets — for Claude CLI
- `GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}` — for GitHub API (ephemeral, repo-scoped)

**Env vars**:
- `CLAUDE_CODE_SKIP_PROMPT_HISTORY=1`
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`

**Steps**:
1. `actions/checkout@v6` with `fetch-depth: 0` at PR head SHA
2. `git fetch origin base_ref`
3. `actions/setup-node@v4` (node 20)
4. `npm install -g @anthropic-ai/claude-code@2.1.119` (pinned)
5. Fetch PR description via `gh pr view` → `/tmp/pr_description.md`
6. **Code Review**: Claude runs `/review` with:
   - `--append-system-prompt-file /tmp/pr_description.md`
   - Optional `.github/claude-review-prompt.md` (doesn't exist yet)
   - `--allowedTools "Bash(git diff*),Bash(git log*),Bash(git show*),Bash(gh api*),Read,Glob,Grep"`
   - `--max-turns 10`
   - Prompt instructs Claude to post inline comments via `gh api repos/.../pulls/.../comments`
   - Prompt instructs Claude to check existing comments to avoid duplicates
   - Output → `/tmp/code_review_output.md`
7. **Post code review comment**: Upsert logic
   - Marker: `<!-- claude-code-review -->`
   - Find existing comment by `github-actions[bot]` with marker → PATCH
   - If none → create new via `gh pr comment`
   - 60KB truncation guard on summary body
8. **Security Review**: Same pattern as code review but with `/security-review` prompt
   - Security inline comments prefixed with lock emoji
   - Marker: `<!-- claude-security-review -->`
9. **Post security review comment**: Same upsert logic, different marker

**Key characteristics**:
- Claude posts inline comments directly during its run via `gh api`
- Deduplication is prompt-based (Claude instructed to check, not enforced)
- Summary comments use HTML marker + upsert (find existing → PATCH, else create)
- Two separate Claude sessions (code + security)
- `if: always()` on posting steps ensures comments even if review step fails

---

### Bitbucket Pipelines (`bitbucket-pipelines.yml`)

**Image**: `node:20.20.2`

**Cache**: `claude-cli` cache keyed on `bitbucket-pipelines.yml` at `/usr/local/lib/node_modules/@anthropic-ai`

**Trigger**: `pull-requests: "**"` (all PRs)

**Auth**:
- `CLAUDE_CODE_OAUTH_TOKEN` — for Claude CLI (pipeline variable, not in YAML)
- `BB_TOKEN` — Bearer token for Bitbucket API

**Pipeline blocks** (sequential, each `- |` block):

1. **Install**: `apt-get install jq`, `npm install -g @anthropic-ai/claude-code@2.1.119`
2. **Env setup**: `BB_API` URL, `BB_AUTH` header (uses `>-` folded scalar for YAML colon safety)
3. **Fetch PR data + skip checks**:
   - Fetch PR JSON from Bitbucket API
   - Skip if draft (`jq -r '.draft // false'`)
   - Skip if `[skip-review]` in title
   - Skip if stale (PR head SHA != pipeline commit)
   - Write `/tmp/skip_review` sentinel file if skipping
   - Write PR description wrapped in `<pr-context>` XML tags → `/tmp/pr_description.md`
4. **Generate diff**:
   - Uses `BITBUCKET_PR_DESTINATION_COMMIT` for diff (fallback to branch ref)
   - POSIX-compatible awk parses diff hunk headers → `/tmp/valid_lines.txt`
   - Maps every valid changed line as `file:linenum`
5. **Fetch prior bot comments**:
   - Paginated fetch from `${BB_API}/comments?pagelen=100`
   - Filters for `[claude-code-review]` or `[claude-security-review]` markers
   - Extracts inline path:line + comment text, or summary first line
   - Writes to `/tmp/prior_comments.md`
6. **Code Review**:
   - Writes prompt to `/tmp/review_prompt.txt` with heredoc
   - Prompt says: "Review ONLY the following PR diff. Do not explore the broader codebase."
   - Includes prior comments dedup instructions
   - Requires JSON output: `{"summary": "...", "comments": [{"path", "line", "severity", "comment"}]}`
   - Severity levels: critical, warning, suggestion, nitpick
   - Appends diff wrapped in `<diff>` tags
   - Appends prior comments wrapped in `<prior-review-comments>` tags (if any)
   - Appends PR description (if any)
   - Pipes to `claude --print --output-format text --allowedTools "..." --max-turns 10`
   - Output → `/tmp/code_review_output.json`
7. **Post code review comments**:
   - Marker: `[claude-code-review]`
   - Extracts JSON (handles markdown fences fallback)
   - **Deletes old comments**: Paginated fetch, delete all with marker
   - **Posts inline comments**: Validates each against `/tmp/valid_lines.txt`
     - If line not in diff → posts as general PR comment with file reference
     - If valid → posts inline with severity icon (red/yellow/blue/white circle)
   - **Posts summary comment**: Overall assessment + finding count
   - **Fallback**: If JSON parse fails, posts raw output as comment
8. **Security Review**: Same pattern as code review but:
   - Marker: `[claude-security-review]`
   - Security-focused prompt (injection, auth bypass, secrets, XSS)
   - Severity levels: critical, warning, suggestion (no nitpick)
9. **Post security review comments**: Same pattern as code review posting

**Key characteristics**:
- Structured JSON output from Claude (not free-form)
- Diff-aware line validation (awk-generated valid line map)
- Wipe-and-replace comment strategy (delete old → post new)
- Prior bot comments fed as context to reduce repetition
- XML tag sandboxing for PR description and prior comments
- Graceful degradation (raw output posted if JSON parse fails)
- All API calls handle pagination
- POSIX-compatible (no gawk dependencies)

---

## 2. Official `anthropics/claude-code-action@v1` Analysis

### Overview
- **Repo**: https://github.com/anthropics/claude-code-action
- **Stars**: 7.3k | **Releases**: 171+ | **Language**: TypeScript 93%
- **Runtime**: Composite action using Bun 1.3.6

### Key Inputs (30+)
| Input | Default | Purpose |
|---|---|---|
| `trigger_phrase` | `@claude` | Keyword to trigger in comments |
| `assignee_trigger` | — | Trigger on issue assignment |
| `label_trigger` | `claude` | Trigger on label add |
| `prompt` | — | Main instruction prompt |
| `claude_args` | — | Additional CLI args |
| `anthropic_api_key` | — | Direct API auth |
| `claude_code_oauth_token` | — | OAuth auth |
| `use_bedrock` | false | Amazon Bedrock |
| `use_vertex` | false | Google Vertex AI |
| `use_foundry` | false | Microsoft Foundry |
| `use_sticky_comment` | false | Single updating comment |
| `classify_inline_comments` | true | Buffer + filter test/probe comments |
| `track_progress` | false | Visual checkboxes |
| `use_commit_signing` | false | GPG/SSH signing |
| `allowed_bots` | — | Bot allowlist |
| `allowed_non_write_users` | — | Non-write user allowlist |
| `include_fix_links` | true | "Fix this" links |
| `display_report` | false | Report display |
| `plugins` | — | Plugin list |
| `plugin_marketplaces` | — | Plugin marketplace URLs |

### Key Outputs
- `execution_file`: Path to execution output
- `branch_name`: Branch created (if any)
- `structured_output`: JSON when `--json-schema` provided
- `session_id`: Resumable session ID

### Features We Don't Have
| Feature | Detail |
|---|---|
| **Interactive `@claude`** | Responds to mentions in PR/issue comments |
| **Code changes/commits** | Can push fixes directly to PR branch |
| **Progress tracking** | Visual checkboxes during execution |
| **Subprocess isolation** | Bubblewrap sandboxing on Linux |
| **Prompt injection sanitization** | Strips HTML comments, invisible chars, hidden attributes |
| **`classify_inline_comments`** | Buffers comments, filters test/probe ones post-session |
| **Multi-cloud auth** | Bedrock, Vertex AI, Foundry |
| **Plugin system** | Plugins + plugin marketplaces |
| **Commit signing** | GPG/SSH |
| **Structured outputs** | JSON schema validation for downstream automation |
| **Session resumption** | `session_id` output for continuing conversations |
| **MCP inline comment tool** | `mcp__github_inline_comment__create_inline_comment` |

### Features It Lacks vs Our Implementation
| Our Feature | Official Action |
|---|---|
| Dual-pass (code + security) as native design | Must configure two separate jobs |
| Explicit skip conditions in `if:` block | Must add manually |
| Cancel-in-progress concurrency | Must add manually |
| Prior bot comment context (Bitbucket) | N/A (GitHub only) |
| Diff-aware line validation (Bitbucket) | Uses MCP tool instead |
| Wipe-and-replace comments (Bitbucket) | Sticky comment model |
| Pinned CLI version | Uses bundled/latest |

### Security Model
- Short-lived repository-scoped tokens (expire when job completes)
- Subprocess env scrubbing (best-effort secret removal)
- Bubblewrap PID-namespace isolation on Linux
- Prompt injection: sanitizes HTML comments, invisible characters, hidden attributes
- Cannot submit formal GitHub PR reviews or approve PRs (by design)
- `allowed_non_write_users` marked as "RISKY" — bypasses write-access requirement
- Warns against personal access tokens (prompt injection recovery risk)

### Limitations
- Cannot submit formal GitHub PR reviews
- Cannot approve pull requests
- Cannot execute arbitrary bash by default
- Cannot do git operations beyond pushing commits
- Single comment model (one updating comment)
- Only accesses current repo context

---

## 3. Consensus Decision: Hybrid Migration

### For GitHub
**Keep policy wrapper, swap execution to official action.**

```
Workflow YAML (ours)
├── if: conditions (draft, skip-review, dependabot, fork)
├── concurrency group (cancel-in-progress)
├── Job 1: Code Review
│   └── uses: anthropics/claude-code-action@<pinned-sha>
│       with prompt for /review
└── Job 2: Security Review
    └── uses: anthropics/claude-code-action@<pinned-sha>
        with prompt for /security-review
```

**Gains**: classify_inline_comments, subprocess isolation, prompt injection sanitization, interactive @claude, progress tracking, MCP inline comment tool, lower maintenance

**Preserves**: dual-pass, skip conditions, concurrency, permissions model, PR description context

**Pin action to commit SHA, not `@v1`.**

### For Bitbucket
**Keep current custom pipeline** — no official Bitbucket action exists.

The Bitbucket pipeline has unique features not available in the official action:
- Structured JSON output parsing
- Diff hunk line validation map
- Wipe-and-replace comment lifecycle
- Prior bot comment context injection
- XML tag sandboxing
- POSIX-compatible awk processing
- Paginated API handling

---

## 4. Known Issues Fixed in Previous Session

These bugs were encountered and fixed in the Bitbucket pipeline:

1. **`awk` POSIX compatibility**: `match()` with 3rd arg is gawk-only. Fixed with `split()`.
2. **`--append-user-prompt-file` doesn't exist**: Not a real flag in claude-code@2.1.119. Fixed by appending content directly into prompt file.
3. **YAML colon-space parsing**: `Authorization: Bearer` parsed as mapping key. Fixed with `>-` folded scalar.

---

## 5. Report Files

Two management-facing reports were created and verified via Codex debate:
- `REPORT_GITHUB.md` — GitHub Actions integration
- `REPORT_BITBUCKET.md` — Bitbucket Pipelines integration

Reports were corrected for:
- "Every PR" → "eligible PRs" with skip conditions
- Removed "no source code leaves" claim (Claude is cloud API)
- Fixed "App Password" → "Bearer token"
- Softened "zero invalid" guarantee
- Dedup/repetition = AI-guided, not deterministic
- Business impact = projected benefits, not proven outcomes
- Prompt injection → "prompt shaping" (XML tags aren't security boundary)

---

## 6. File Inventory

| File | Platform | Purpose |
|---|---|---|
| `.github/workflows/claude-review.yml` | GitHub | Current custom workflow |
| `bitbucket-pipelines.yml` | Bitbucket | Current custom pipeline |
| `REPORT_GITHUB.md` | — | Management report (GitHub) |
| `REPORT_BITBUCKET.md` | — | Management report (Bitbucket) |
| `.github/claude-review-prompt.md` | GitHub | Optional custom prompt (doesn't exist yet) |

---

## 7. Auth Token Summary

| Token | Platform | Purpose | Storage |
|---|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Both | Claude CLI authentication | GitHub Secrets / BB Pipeline Variables |
| `GITHUB_TOKEN` | GitHub | GitHub API (PR comments, inline comments) | Auto-provided by GitHub Actions |
| `BB_TOKEN` | Bitbucket | Bitbucket API (PR comments, inline comments) | BB Pipeline Variables |
