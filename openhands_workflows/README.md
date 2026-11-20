# OpenHands Workflows

This folder is a personal, repository-neutral workflow runner built on top of
the OpenHands SDK. It is intentionally separate from `openhands-sdk/`,
`openhands-tools/`, and `openhands-workspace/` so it can later move into its own
repository without changing the SDK packages.

## Shared issue flows

The caller selects a flow by adding one of these labels to an Issue:

| Label | Stages | Delivery |
| --- | --- | --- |
| `agent:openhands-full-development` | Inspect → Proposal → Claude plan review → Human approval → Implement → Verify → Claude code review | Pull Request |
| `agent:openhands-backfix` | Investigate → Implement → Verify → Independent Claude review | Pull Request |
| `agent:openhands-quick-fix` | Implement → Verify | Pull Request |
| `agent:openhands-implementation-only` | Implement → Verify | Patch artifact |

Full-development uses two GitHub events and stores its proposal in a marked
Issue comment: apply the label to generate a read-only proposal, then reply
`/openhands approve` (optionally followed by conditions) to implement it. This
keeps the approval checkpoint simple without a separate workflow database.
Review-only reporting and custom stage ordering are not implemented yet.

The flows all use the same runner and security gates. The default verification
command is `make check`; a caller can override it with `OPENHANDS_CHECK_COMMAND`
when a repository uses another command. The Issue title and body are treated as
untrusted task input. OpenHands works in an isolated Docker workspace mounted to
the target repository checkout. The host keeps publication credentials away
from the agent container.

## Reusable GitHub workflow

`.github/workflows/openhands-issue.yml` is a reusable workflow. A project only
needs a small caller workflow that listens for Issue labels and calls this
workflow. Flow logic and runner code remain here. The caller may choose a
verification command or runner label, but does not define its own flow.

The caller is the same small adapter in every repository; omit `check_command`
when the repository follows the shared `make check` convention:

```yaml
on:
  issues:
    types: [labeled]
  issue_comment:
    types: [created]

permissions:
  contents: read
  issues: write
  pull-requests: read

jobs:
  issue-flow:
    if: github.event_name == 'issues' && contains(fromJSON('["agent:openhands-backfix","agent:openhands-quick-fix","agent:openhands-implementation-only"]'), github.event.label.name)
    uses: slcosentino1/software-agent-sdk/.github/workflows/openhands-issue.yml@main
    with:
      flow_label: ${{ github.event.label.name }}
    secrets:
      OPENHANDS_LLM_API_KEY: ${{ secrets.OPENHANDS_LLM_API_KEY }}
      OPENHANDS_PUBLISH_TOKEN: ${{ secrets.OPENHANDS_PUBLISH_TOKEN }}

  full-development-proposal:
    if: github.event_name == 'issues' && github.event.label.name == 'agent:openhands-full-development'
    uses: slcosentino1/software-agent-sdk/.github/workflows/openhands-issue.yml@main
    with:
      flow_label: agent:openhands-full-development
      phase: proposal
    secrets:
      OPENHANDS_LLM_API_KEY: ${{ secrets.OPENHANDS_LLM_API_KEY }}

  full-development-implementation:
    if: github.event_name == 'issue_comment' && startsWith(github.event.comment.body, '/openhands approve')
    uses: slcosentino1/software-agent-sdk/.github/workflows/openhands-issue.yml@main
    with:
      flow_label: agent:openhands-full-development
      phase: implementation
    secrets:
      OPENHANDS_LLM_API_KEY: ${{ secrets.OPENHANDS_LLM_API_KEY }}
      OPENHANDS_PUBLISH_TOKEN: ${{ secrets.OPENHANDS_PUBLISH_TOKEN }}
```

The called workflow uses its repository-scoped `GITHUB_TOKEN` to post Issue
comments and publish branches and PRs. The caller must grant `contents: write`,
`issues: write`, and `pull-requests: write`; enable GitHub Actions to create Pull
Requests in repository settings. An optional `llm_env_file` input lets a
self-hosted runner load `LLM_API_KEY`, `LLM_BASE_URL`, and `LLM_MODEL` from a
runner-local file. Otherwise it accepts `OPENHANDS_LLM_API_KEY` as a secret. The
runner also needs the Aily CLI authenticated for `aily claude` for Full-development
proposal and code reviews and for Backfix. A shared runner must be registered at
organization level or otherwise be available to every caller repository.

The default model settings are `openai/us.openai.gpt-6-astra`,
`https://litellm.shared.aily-app.com`, and
`ghcr.io/openhands/agent-server:latest-python`. Set repository or organization
Actions variables `OPENHANDS_LLM_MODEL`, `OPENHANDS_LLM_BASE_URL`, and
`OPENHANDS_AGENT_SERVER_IMAGE` to override them. Rotate any provider key that
has previously appeared in a terminal log or chat before saving it as a secret.

Each caller repository needs the four labels and either the `OPENHANDS_LLM_API_KEY`
secret or a readable runner-local env file. No long-lived publish token is
required; the workflow token is scoped to the target repo and current run.

The current AskDoc runner is repository-scoped. GitHub makes self-hosted runners
available at repository, organization, or enterprise scope; to share one Mac
across organization repos, register it at organization scope with the
`openhands-agent` label. For personal-account repos, either register a runner
installation for each repo or move the repos into an organization. A project can
keep a repo-specific runner label in its caller workflow if needed.

This is a personal harness, not a change to the upstream OpenHands SDK API.
