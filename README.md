DeepSeek PR Review Action
=========================

Composite action that posts an AI code review on pull requests using DeepSeek, with support for comment-triggered reruns.

Usage
-----

Create `.github/workflows/deepseek-review.yml` in your repo:

    name: DeepSeek Review
    on:
      pull_request:
        types: [opened, synchronize, reopened]
      issue_comment:
        types: [created]
    permissions:
      contents: read
      pull-requests: write
    jobs:
      review:
        runs-on: ubuntu-latest
        steps:
          - uses: zuzak/deepseek-review-action@v1
            with:
              deepseek-api-key: ${{ secrets.DEEPSEEK_API_KEY }}

Add your own prompt by creating `.github/deepseek/prompt.md`. If no file exists, the bundled generic template is used.

Inputs
------

| Input | Required | Default | Description |
|---|---|---|---|
| `deepseek-api-key` | yes | — | DeepSeek API key |
| `github-token` | no | `github.token` | Token for posting reviews. Swap to a GitHub App token to enable proper re-request flow (#688). |
| `reviewer-login` | no | `github-actions[bot]` | GitHub login that posts reviews; used to dismiss stale CHANGES_REQUESTED. Update to your App bot login when using an App token. |
| `prompt-template-glob` | no | `.github/deepseek/*.md` | Glob for prompt files, concatenated in sort order |
| `exclude-paths` | no | _(empty)_ | Newline-separated gitignore-style paths excluded from the HEAD snapshot |
| `skip-if-paths-changed` | no | _(empty)_ | Skip review if PR touches any of these paths |
| `recheck-comment-prefix` | no | `/ds-recheck` | Comment prefix (any author) that triggers a recheck |
| `max-rechecks` | no | `2` | Max recheck comments per DS round before deadlock notice |
| `label-deadlock` | no | _(empty)_ | Label to apply on recheck cap or IMPASSE. Leave empty to disable. |
| `model` | no | `deepseek-reasoner` | DeepSeek model |

Prompt template
---------------

The prompt template is assembled by concatenating all files matching `prompt-template-glob` in sort order. Use multiple files to compose sections (e.g. `00-instructions.md`, `10-repo-context.md`).

Placeholders substituted at runtime:

| Placeholder | Content |
|---|---|
| `{{TRIGGER}}` | `first-or-push` or `recheck` |
| `{{PR_LABELS}}` | Comma-separated PR labels |
| `{{PRIOR_REVIEW_THREAD}}` | All PR comments in chronological order |
| `{{REPO_CONTENTS}}` | Full HEAD snapshot (minus `exclude-paths`) |
| `{{PATCH}}` | `git diff` against the base branch |

Verdict markers
---------------

The model ends its response with an optional marker:

- `<!-- APPROVE -->` — posted as a GitHub Approved review
- `<!-- REQUEST CHANGES -->` — posted as a Request Changes review; blocks merge
- `<!-- IMPASSE -->` — posted as a plain comment; prior review state preserved; deadlock label applied if configured
- _(none)_ — posted as a COMMENTED review

Recheck cap
-----------

After `max-rechecks` `/ds-recheck` comments in a single DS round, the action posts a notice and applies the deadlock label (if configured). A new push starts a fresh round with a full recheck budget.

Migrating from an inline workflow
----------------------------------

If you have an existing inline workflow (e.g. the one previously in `zuzak/claude`):

1. Replace the workflow body with the `uses:` step above.
2. Move your prompt from `deepseek/prompt.md` to `.github/deepseek/prompt.md`.
3. Add `exclude-paths` and `skip-if-paths-changed` as needed.
4. Remove any zuzak/claude-specific label inputs unless your repo uses the same labels.
