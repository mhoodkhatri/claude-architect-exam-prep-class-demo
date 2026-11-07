# CI Notes

Notes on the GitHub Actions workflows in `.github/workflows/`.

## Workflows

| Workflow | File | Trigger | Purpose |
|----------|------|---------|---------|
| Markdown only | `markdown-only.yml` | Push to any branch; PRs targeting any branch | Fails if any tracked file is not Markdown or `.txt` |
| Claude Code | `claude.yml` | `@claude` in issue/PR comments, review comments, reviews, or issue title/body; issue assigned | Runs Claude Code on request |
| Claude Code Review | `claude-code-review.yml` | PR opened, synchronized, reopened, or marked ready for review | Automated PR review posted as inline comments |

## Markdown only

This repo is meant to hold only Markdown notes. The job checks out the code and runs `git ls-files`, drops `.md` and `.txt` files, then drops an allowlist of exceptions. Any file left over fails the job with an `::error` annotation.

Allowed non-Markdown files:

- `.gitignore`
- `.gitattributes`
- `.claude/settings.json`
- `.github/workflows/*.yml` / `*.yaml`

To allow another file type, edit the `allowed` regex in `markdown-only.yml`.

Permissions: `contents: read` only.

## Claude Code (`claude.yml`)

- Runs only when the event text contains `@claude`.
- Uses `anthropics/claude-code-action@v1`, authenticated with the `CLAUDE_CODE_OAUTH_TOKEN` repository secret.
- Permissions: `contents`, `pull-requests`, `issues` read, `actions: read` (so Claude can read CI results), and `id-token: write`.
- Claude cannot modify files under `.github/workflows/`; the GitHub App does not have that permission.
- Anything Claude pushes must still pass the Markdown-only check, so it can only add Markdown files (like this one).

## Claude Code Review (`claude-code-review.yml`)

- Runs the `code-review@claude-code-plugins` plugin on each PR.
- Only tool allowed: `mcp__github_inline_comment__create_inline_comment`.
- Uses the same `CLAUDE_CODE_OAUTH_TOKEN` secret.
- Optional `paths:` and author filters are present as comments, not enabled.

## Setup checklist

- [ ] `CLAUDE_CODE_OAUTH_TOKEN` secret is set under Settings → Secrets and variables → Actions
- [ ] Claude GitHub App is installed on the repo
- [ ] Consider making "Markdown only / check-file-types" a required status check on `main`

## Troubleshooting

- **Markdown check fails:** read the `Not a Markdown file` annotation, then either rename or remove the file, or add it to the allowlist.
- **Claude doesn't respond:** confirm the comment contains `@claude` and the secret exists.
- **Claude can't edit a workflow:** expected; make workflow changes by hand.
