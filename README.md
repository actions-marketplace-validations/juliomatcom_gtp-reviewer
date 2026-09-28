# gtp-reviewer

A composite action that reviews a pull request with Codex and posts the result:

- Inline findings, each ending with its severity (Critical, Major or Minor).
- A summary with a 🟢/🟡/🔴 merge confidence. Confidence follows the findings: 🟢 means no open finding above Minor.
- A warning in the summary listing what neither the tests nor Codex could verify, such as a workflow that only runs after merge. It never lowers confidence.
- An approval on 🟢. The action dismisses its own earlier approval when a later push rates lower.

Each run feeds the PR's earlier review threads into the prompt. It only counts replies from people with write access, and it tells Codex not to raise a finding again once it is resolved or answered.

The review is also exposed as the `review` output: JSON matching [`review-schema.json`](src/review-schema.json).

## Inputs

| Input                | Default               | Notes                                                                                                           |
| -------------------- | --------------------- | --------------------------------------------------------------------------------------------------------------- |
| `openai-api-key`     | required              |                                                                                                                 |
| `model`              | `gpt-6-luna`          |                                                                                                                 |
| `effort`             | `medium`              |                                                                                                                 |
| `permission-profile` | `:workspace`          | Codex sandbox profile.                                                                                          |
| `instructions-file`  | none                  | Repo-relative project instructions, read from the **base branch** and appended to [`review.md`](src/review.md). |
| `github-token`       | `${{ github.token }}` | Reads earlier review threads and posts the review; needs `pull-requests: write`.                                |

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
  pull-requests: write

jobs:
  review:
    # pull_request_target hands secrets to forks; never review them.
    if: github.event.pull_request.draft == false && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: juliomatcom/gtp-reviewer@main
        with:
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          instructions-file: .github/codex/review.md
```

To let the bot approve, the repo needs an `OPENAI_API_KEY` secret and the setting _Allow GitHub Actions to create and approve pull requests_. Without that setting, the review is posted as a comment instead.
