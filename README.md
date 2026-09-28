# gtp-reviewer

A composite action that reviews a pull request with Codex and posts the result:

- Inline findings, each ending with its severity (Critical, Major or Minor).
- A summary with a 🟢/🟡/🔴 merge confidence. Confidence follows the findings: 🟢 means no open finding above Minor.
- A warning in the summary listing what neither the tests nor Codex could verify, such as a workflow that only runs after merge. It never lowers confidence.
- A footer with the run's token usage and estimated cost, read from the Codex session log and priced from a short-context table in [`usage.js`](src/usage.js). Models missing from the table show tokens only.
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

## Example output

A PR with one minor finding gets 🟢 High, so the action approves it. It posts the finding as an inline comment on `src/cache.ts:42`:

```markdown
**Expired entries are never evicted.**

`get` returns `undefined` for an expired key but leaves it in the map, so memory grows with every unique key. Delete the entry when it is found expired.

Severity: Minor
```

And this summary as the review body:

```markdown
## Codex review

### Confidence

🟢 High

The new cache is covered by unit tests for hits, misses and expiry; the only finding is minor.

> [!WARNING]
> **Not verified.** Check before or right after merging:
>
> - The cache is not exercised under real traffic; watch memory after deploy.

<sub>gpt-6-luna (medium) · 120.4k input, 98.1k cached · 5.2k output · ≈ $0.0058</sub>
```

With no findings, the summary says `No findings.` above the confidence. When an inline comment is rejected, for example because its line is outside the diff, each finding is listed in the summary instead: `` - `src/cache.ts:42` **Expired entries are never evicted.** — … — Severity: Minor ``.
