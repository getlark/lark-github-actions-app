# Lark GitHub Actions

GitHub composite action that proposes end-to-end [Lark](https://getlark.ai) tests for each pull request and creates them on demand. Drop it into a workflow and Lark will draft suggestions, post them as a PR comment, and — when a maintainer replies `/create-workflows` — materialize the approved subset as real Lark workflows.

Under the hood, Claude Code reads the diff and discovers your existing coverage by calling the [Lark MCP server](https://docs.getlark.ai/mcp-quickstart) — searching workflows by name and inspecting their steps before proposing anything new, then using the same MCP server's `create_workflow` tool to materialize each approved proposal.

## How it works

The action runs in two phases off the same `getlark/lark-github-actions-app@v1` step — it branches internally on `github.event_name`:

1. **Propose** (on `pull_request` open/reopen) — Claude drafts test proposals, the action posts them as a PR comment with a hidden JSON marker, and labels the PR `lark:tests-proposed`.
2. **Create** (on `issue_comment` with body `/create-workflows` or `/create-workflows 1 3 …`) — only repo collaborators with `write`, `maintain`, or `admin` permission can trigger this. The action re-reads the most recent proposals comment, filters by the listed 1-based indices (or takes all of them), and asks Claude to create one Lark workflow per entry. A new PR comment lists the created workflow ids and dashboard links.

## Usage

```yaml
# .github/workflows/lark.yml
name: Lark — propose and create tests

on:
  pull_request:
    types: [opened, reopened]
  issue_comment:
    types: [created]

permissions:
  id-token: write
  pull-requests: write
  contents: read

jobs:
  lark:
    # Skip PRs from forks — they cannot mint OIDC tokens with our trusted audience.
    # For comment events we additionally guard on `issue.pull_request != null` so
    # the job never runs on plain issue comments.
    if: >-
      (github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository) ||
      (github.event_name == 'issue_comment' && github.event.issue.pull_request != null)
    runs-on: ubuntu-latest
    steps:
      - uses: getlark/lark-github-actions-app@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `anthropic-api-key` | yes | — | Anthropic API key used by `claude-code-action` to draft proposals. Pass via `${{ secrets.ANTHROPIC_API_KEY }}` so GitHub masks it in logs. |
| `lark-api-base-url` | no | `https://api.getlark.ai` | Override to point at staging. Production customers should leave the default. |

## Required workflow permissions

| Permission | Why |
| --- | --- |
| `id-token: write` | Mint the OIDC token that Lark exchanges for a scoped API token. |
| `pull-requests: write` | Post the proposals/results comments, manage the `lark:*` labels, and add reactions to `/create-workflows` comments. |
| `contents: read` | Check out the PR head so Claude Code can diff against the base. |

## Releases & tagging convention

This action follows the GitHub Actions standard floating-major-tag pattern:

- **Immutable point releases**: every release gets an immutable annotated tag, e.g. `v1.0.0`, `v1.0.1`, `v1.1.0`.
- **Floating major tag**: `v1` is a lightweight tag that is **force-moved** to the latest `v1.x.x` commit on every release. Customers pin `getlark/lark-github-actions-app@v1` and automatically pick up patch and minor releases.
- **Breaking changes** bump the major tag — `v2.0.0` ships alongside a new floating `v2` tag. The old `v1` keeps pointing at the last `v1.x.x` so existing customer workflows do not break.

### Cutting a release

```bash
# 1. Tag the new immutable version.
git tag -a v1.0.1 -m "v1.0.1"
git push origin v1.0.1

# 2. Re-point the floating major tag.
git tag -f v1 v1.0.1
git push origin v1 --force
```

Always create the immutable tag first and push it before moving the floating tag — that way the floating tag never points at a commit that is not also reachable from a permanent tag.
