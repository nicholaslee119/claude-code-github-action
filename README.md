# Claude Code GitHub Action

This GitHub Action integrates Claude Code with GitHub workflows, enabling automated code changes based on issues and PR comments.

## Features

- Automatically process issues with a specific label using Claude Code
- Generate code changes and create PRs with detailed summaries
- Process PR review comments and update code accordingly
- Support for both AWS Bedrock and Anthropic API

## Prerequisites

To use this Action, you need:

1. Claude Code CLI (automatically installed by the Action)
2. Necessary credentials:
   - For AWS Bedrock: `BEDROCK_AWS_ACCESS_KEY_ID` and `BEDROCK_AWS_SECRET_ACCESS_KEY`
   - For Anthropic API: `ANTHROPIC_API_KEY`
   - GitHub token: `GITHUB_TOKEN` (automatically provided) or a custom PAT token

## Usage

### Processing Issues

```yaml
name: Claude Code Issue Handler

on:
  issues:
    types: [opened, edited, labeled]

permissions:
  issues: write
  contents: write
  pull-requests: write

jobs:
  process-issue:
    runs-on: ubuntu-latest
    if: contains(github.event.issue.labels.*.name, 'claude-code')
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Process issue with Claude Code
        uses: nicholaslee119/claude-code-github-action@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          BEDROCK_AWS_ACCESS_KEY_ID: ${{ secrets.BEDROCK_AWS_ACCESS_KEY_ID }}
          BEDROCK_AWS_SECRET_ACCESS_KEY: ${{ secrets.BEDROCK_AWS_SECRET_ACCESS_KEY }}
          # If using Anthropic API instead of Bedrock:
          # ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          issue-number: ${{ github.event.issue.number }}
          issue-body: ${{ github.event.issue.body }}
          issue-title: ${{ github.event.issue.title }}
          base-branch: 'main'  # Adjust according to your repository
          aws-region: 'us-east-2'
          model-id: 'anthropic.claude-3-7-sonnet-20250219-v1:0'
          use-bedrock: 'true'
          cross-region-inference: 'true'
```

### Processing PR Comments

```yaml
name: Claude Code PR Review Handler

on:
  issue_comment:
    types: [created]

permissions:
  issues: write
  contents: write
  pull-requests: write

jobs:
  process-review:
    runs-on: ubuntu-latest
    if: github.event.issue.pull_request && startsWith(github.event.comment.body, 'Review:')
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Get PR details
        id: pr
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const prNumber = context.payload.issue.number;
            const commentId = context.payload.comment.id;
            const reviewFeedback = context.payload.comment.body.substring(7).trim();
            
            const { data: pr } = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: prNumber
            });
            
            core.setOutput('number', prNumber);
            core.setOutput('commentId', commentId);
            core.setOutput('feedback', reviewFeedback);
            core.setOutput('headRef', pr.head.ref);
      
      - name: Process PR review with Claude Code
        uses: nicholaslee119/claude-code-github-action/pr-review@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          BEDROCK_AWS_ACCESS_KEY_ID: ${{ secrets.BEDROCK_AWS_ACCESS_KEY_ID }}
          BEDROCK_AWS_SECRET_ACCESS_KEY: ${{ secrets.BEDROCK_AWS_SECRET_ACCESS_KEY }}
        with:
          pr-number: ${{ steps.pr.outputs.number }}
          comment-id: ${{ steps.pr.outputs.commentId }}
          feedback: ${{ steps.pr.outputs.feedback }}
          head-ref: ${{ steps.pr.outputs.headRef }}
```

## Input Parameters

### Main Action (Issue Processing)

| Parameter | Description | Required | Default |
|-----------|-------------|----------|---------|
| `issue-number` | The issue number to process | Yes | - |
| `issue-body` | The body content of the issue | Yes | - |
| `issue-title` | The title of the issue | Yes | - |
| `branch-name` | The name of the branch to create | No | `claude-code/issue-{issue-number}` |
| `base-branch` | The base branch to create PR against | No | `main` |
| `aws-region` | AWS region for Bedrock | No | `us-east-2` |
| `model-id` | Claude model ID to use | No | `anthropic.claude-3-7-sonnet-20250219-v1:0` |
| `use-bedrock` | Whether to use AWS Bedrock (true) or Anthropic API (false) | No | `true` |
| `cross-region-inference` | Whether to enable cross-region inference for Bedrock | No | `true` |
| `allowed-tools` | Tools to allow Claude Code to use | No | `Bash(git diff:*) Bash(git log:*) Edit` |

### PR Review Action

| Parameter | Description | Required | Default |
|-----------|-------------|----------|---------|
| `pr-number` | The PR number to process | Yes | - |
| `comment-id` | The ID of the comment to process | Yes | - |
| `feedback` | The feedback content from the comment | Yes | - |
| `head-ref` | The head ref of the PR | Yes | - |
| `aws-region` | AWS region for Bedrock | No | `us-east-2` |
| `model-id` | Claude model ID to use | No | `anthropic.claude-3-7-sonnet-20250219-v1:0` |
| `use-bedrock` | Whether to use AWS Bedrock (true) or Anthropic API (false) | No | `true` |
| `cross-region-inference` | Whether to enable cross-region inference for Bedrock | No | `true` |
| `allowed-tools` | Tools to allow Claude Code to use | No | `Bash(git diff:*) Bash(git log:*) Edit` |

## Outputs

### Main Action (Issue Processing)

| Output | Description |
|--------|-------------|
| `pr-number` | The number of the created PR |
| `summary` | Summary of changes made by Claude Code |

## How It Works

1. **Issue Processing**:
   - When an issue with the `claude-code` label is created or updated
   - Claude Code analyzes the issue content
   - Code changes are generated based on the issue description
   - A PR is created with a detailed summary of changes

2. **PR Review Processing**:
   - When a comment starting with `Review:` is added to a PR
   - Claude Code analyzes the feedback
   - Code is updated based on the feedback
   - A comment is added to the PR confirming the update

## Environment Variables

- `GITHUB_TOKEN`: GitHub token for authentication (automatically provided)
- `BEDROCK_AWS_ACCESS_KEY_ID`: AWS access key ID for Bedrock
- `BEDROCK_AWS_SECRET_ACCESS_KEY`: AWS secret access key for Bedrock
- `ANTHROPIC_API_KEY`: Anthropic API key (if not using Bedrock)

## AWS Bedrock vs. Anthropic API

This action supports two methods for accessing Claude models:

### AWS Bedrock

- Set `use-bedrock: 'true'`
- Requires AWS credentials
- Supports cross-region inference
- Data stays within AWS environment
- Good for enterprise environments with existing AWS infrastructure

### Anthropic API

- Set `use-bedrock: 'false'`
- Requires Anthropic API key
- Simpler setup
- Direct access to the latest model versions
- May provide lower latency depending on your location

## License

MIT
