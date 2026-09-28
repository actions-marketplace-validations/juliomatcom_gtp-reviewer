# gtp-reviewer

Codex pull request review as two composite actions:

- `juliomatcom/gtp-reviewer` checks out the PR, builds the prompt and runs Codex. Output `review` is JSON matching [`review-schema.json`](review-schema.json): inline findings plus a high/medium/low merge confidence.
- `juliomatcom/gtp-reviewer/post` posts that result as a PR review: inline comments, an approval on high confidence, and its own earlier approval dismissed when a later push rates lower.

They are separate so Codex never runs in a job holding `pull-requests: write`.

Each run feeds the PR's earlier review threads into the prompt. It only counts replies from people with write access, and it tells Codex not to raise a finding again once it is resolved or answered.

## Inputs

`juliomatcom/gtp-reviewer`:

| Input                | Default               | Notes                                                                                |
| -------------------- | --------------------- | ------------------------------------------------------------------------------------ |
| `openai-api-key`     | required              |                                                                                      |
| `model`              | `gpt-6-luna`          |                                                                                      |
| `effort`             | `medium`              |                                                                                      |
| `permission-profile` | `:workspace`          | Codex sandbox profile.                                                               |
| `instructions-file`  | none                  | Repo-relative project instructions, read from the **base branch** and appended to [`review.md`](review.md). |
| `github-token`       | `${{ github.token }}` | Reads earlier review threads.                                                        |

`juliomatcom/gtp-reviewer/post`: `review` (required), `github-token` (default `${{ github.token }}`).

## Usage

```yaml
name: Codex review

on:
  # Runs the workflow from the base branch, so a PR cannot rewrite its own review.
  pull_request_target:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main]

concurrency:
  group: codex-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: read

jobs:
  review:
    # pull_request_target hands secrets to forks; never review them.
    if: github.event.pull_request.draft == false && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    outputs:
      review: ${{ steps.review.outputs.review }}
    steps:
      - id: review
        uses: juliomatcom/gtp-reviewer@main
        with:
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          instructions-file: .github/codex/review.md

  comment:
    needs: review
    if: needs.review.outputs.review != ''
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: juliomatcom/gtp-reviewer/post@main
        with:
          review: ${{ needs.review.outputs.review }}
```

To let the bot approve, the repo needs an `OPENAI_API_KEY` secret and the setting _Allow GitHub Actions to create and approve pull requests_. Without that setting, the review is posted as a comment instead.
