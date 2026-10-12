# interplay-adversarial/.github

Shared, reusable GitHub Actions workflows for the org's repositories. This repo is public so repositories (including other owners') can call them.

## `pr-agent.yml`: AI pull-request review

Runs [PR-Agent](https://github.com/the-pr-agent/pr-agent) (pinned by commit SHA) on a hosted runner and calls an OpenAI-compatible model endpoint.

Inputs: `model`, `api_base`, `max_tokens` (defaults: `qwen38-27b`, its MaaS per-model URL, 64000). `gemma4` is also served (131k context) but untested; use `with: {model: gemma4, api_base: .../prelude-maas/gemma4/v1}`. `glm-53-flash` fails: its reasoning leaks into the output and PR Agent cannot parse it. Secret: `MAAS_API_KEY`.

Caller (`.github/workflows/pr-agent.yml` in a repo):

```yaml
name: PR Agent
on:
  pull_request:
    types: [opened, reopened, ready_for_review, review_requested]
  issue_comment:
    types: [created]

permissions: {}

jobs:
  pr_agent:
    if: >-
      github.event.sender.type != 'Bot' && (
        (github.event_name == 'pull_request'
          && github.event.pull_request.head.repo.full_name == github.repository
          && !github.event.pull_request.draft) ||
        (github.event_name == 'issue_comment'
          && github.event.issue.pull_request
          && startsWith(github.event.comment.body, '/')
          && contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.comment.author_association))
      )
    permissions:
      issues: write
      pull-requests: write
      contents: read
    uses: interplay-adversarial/.github/.github/workflows/pr-agent.yml@main
    secrets:
      MAAS_API_KEY: ${{ secrets.MAAS_API_KEY }}
```

The caller's `if` is the security gate: no fork PRs, no bots, slash commands only from owners, members and collaborators.
