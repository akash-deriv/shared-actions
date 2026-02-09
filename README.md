## Shared Actions Repository

This repository is dedicated to hosting reusable GitHub Actions YAML files that can be shared across different repositories. Centralizing common actions, to promote consistency and efficiency in workflows.

---

## 📚 DocSync AI - Intelligent Documentation Sync

DocSync AI automatically keeps your documentation in sync with your code changes using Claude AI.

### Features

#### 1. **Automatic Documentation Updates (on PR merge)**
When a PR is merged, DocSync AI:
- Analyzes the code changes
- Determines if documentation needs updates
- Automatically updates README.md or CLAUDE.md
- Creates a new PR with the documentation changes

#### 2. **Interactive Documentation Updates (via PR comments)**
On any DocSync-generated PR, team members can request specific changes by commenting:

```
docsync: Add a troubleshooting section for common errors
```

The AI will:
- Review the suggestion with full context
- Make the final decision on what changes to apply
- Commit updates directly to the DocSync PR
- Provide quality control and sanitization

### Setup Instructions

#### 1. **Create Required Secrets**
Go to your repository Settings > Secrets and variables > Actions and add:

| Secret Name | Description | How to Get |
|-------------|-------------|------------|
| `DOCSYNC_GITHUB_TOKEN` | Personal Access Token with `repo` and `workflow` scopes | [Create token](https://github.com/settings/tokens/new) |
| `DOCSYNC_ANTHROPIC_API_KEY` | Anthropic API key for Claude AI | [Get API key](https://console.anthropic.com/) |
| `DOCSYNC_SLACK_WEBHOOK` | (Optional) Slack webhook for notifications | [Create webhook](https://api.slack.com/messaging/webhooks) |

#### 2. **Add Workflow to Your Repository**
Create `.github/workflows/docsync-ai.yml` in your repository:

```yaml
name: 📚 DocSync AI Documentation

on:
  pull_request:
    types: [closed]
    branches:
      - master  # Change to 'main' if needed

  issue_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write

jobs:
  docsync-on-merge:
    if: github.event_name == 'pull_request' && github.event.pull_request.merged == true
    uses: akash-deriv/shared-actions/.github/workflows/docsync-ai.yml@master
    with:
      trigger_type: 'merge'
      base_branch: 'master'
      pr_labels: 'documentation,automated,docsync-ai'
    secrets:
      DOCSYNC_GITHUB_TOKEN: ${{ secrets.DOCSYNC_GITHUB_TOKEN }}
      DOCSYNC_ANTHROPIC_API_KEY: ${{ secrets.DOCSYNC_ANTHROPIC_API_KEY }}
      DOCSYNC_SLACK_WEBHOOK: ${{ secrets.DOCSYNC_SLACK_WEBHOOK }}

  docsync-on-comment:
    if: |
      github.event_name == 'issue_comment' &&
      github.event.issue.pull_request != null &&
      startsWith(github.event.comment.body, 'docsync:')
    uses: akash-deriv/shared-actions/.github/workflows/docsync-ai.yml@master
    with:
      trigger_type: 'comment'
      base_branch: 'master'
      comment_body: ${{ github.event.comment.body }}
      comment_pr_number: ${{ github.event.issue.number }}
    secrets:
      DOCSYNC_GITHUB_TOKEN: ${{ secrets.DOCSYNC_GITHUB_TOKEN }}
      DOCSYNC_ANTHROPIC_API_KEY: ${{ secrets.DOCSYNC_ANTHROPIC_API_KEY }}
```

### Usage

#### Automatic Mode (PR Merge)
Simply merge your PR - DocSync AI will automatically analyze changes and create a documentation PR if needed.

#### Interactive Mode (PR Comments)
On any DocSync AI PR, use the **@docbot command interface** for controlled documentation updates:

**🎯 Command Examples:**
```
@docbot update readme
@docbot clarify auth section
@docbot add example for API authentication
@docbot expand installation steps
@docbot fix typo in configuration section
@docbot revert last change
```

**Legacy format (still supported):**
```
docsync: Add installation instructions for Windows users
```

**Features:**
- ✨ **Structured commands** - Clear, predictable interface (`@docbot <action> <target>`)
- 🎯 **Specific actions** - Update, clarify, add examples, expand, fix, or revert
- 🔄 **Revert capability** - Undo changes with `@docbot revert last change`
- 📋 **Auto-guide** - Each DocSync PR includes a comment with all available commands
- 🤖 **AI-powered** - Claude analyzes context and applies changes intelligently

**Important:**
- Commands MUST start with `@docbot` (case-insensitive)
- Only works on DocSync-generated PRs (with `docsync-ai` or `automated` labels)
- AI makes the final decision - may reject, modify, or improve suggestions
- Provides quality control and sanitization of user input
- All changes are committed automatically and reviewable before merge

### How It Works

1. **Merge Trigger**: When PR merges → AI analyzes diff → Creates doc PR if significant changes detected → Posts @docbot usage guide
2. **Comment Trigger**: When user comments `@docbot <command>` → AI parses command → Applies changes → Commits to PR
3. **AI Decision Making**: Claude AI evaluates all suggestions and makes expert decisions on documentation quality
4. **Safety**: Validates PRs, sanitizes input, only modifies documentation files
5. **Revert Support**: Can undo changes with `@docbot revert last change` for easy rollback

See [example-docsync-workflow.yml](example-docsync-workflow.yml) for a complete configuration example with detailed comments.

---

## Other Actions

#### Example Usage

```yaml
- name: Post preview build comment
  id: post_preview_build_comment
  uses: "deriv-com/shared-actions/.github/actions/post_preview_build_comment@master"
  with:
    issue_number: ${{steps.pr_information.outputs.issue_number}}
    head_sha: ${{github.event.workflow_run.head_sha}}
```
