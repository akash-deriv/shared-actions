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
On any DocSync-generated PR, team members can request specific changes using the **@docbot command interface**:

```
@docbot update readme
@docbot clarify auth section
@docbot add example for API authentication
@docbot expand installation steps
@docbot fix typo in configuration section
@docbot revert last change
```

The AI will:
- Review the command with full context
- Make the final decision on what changes to apply
- Commit updates directly to the DocSync PR
- Provide quality control and sanitization
- Support revert capability for easy rollback

### Setup Instructions

#### 1. **Create Required Secrets**

**In your repository** (Settings > Secrets and variables > Actions), add:

| Secret Name | Description | How to Get |
|-------------|-------------|------------|
| `DOCSYNC_GITHUB_TOKEN` | Personal Access Token with `repo` and `workflow` scopes | [Create token](https://github.com/settings/tokens/new) |
| `DOCSYNC_ANTHROPIC_API_KEY` | Anthropic API key for Claude AI | [Get API key](https://console.anthropic.com/) |

**Optional: Slack Notifications**

DocSync AI supports two methods for Slack notifications:

**Method 1: OAuth Token (Recommended)**
- Stores OAuth token in shared-actions repo (more secure, centralized)
- Each consuming repo provides channel ID and user ID
- Better control and user mentions

**Method 2: Webhook (Legacy)**
- Each repo stores its own webhook URL
- Simpler setup but less flexible

**Setup for OAuth Method:**

1. **In shared-actions repo** (this repo), add secret:
   - `SLACK_OAUTH_TOKEN` - Slack Bot OAuth token with `chat:write` scope ([Create Slack App](https://api.slack.com/apps))

2. **In your consuming repo**, get Slack IDs:
   - **Channel ID**: Open Slack → Right-click channel → View channel details → Copy ID (starts with `C`)
   - **User ID**: Click on user profile → More → Copy member ID (starts with `U`)

3. Pass these as inputs in your workflow (see below)

**Setup for Webhook Method (Legacy):**
- Add `DOCSYNC_SLACK_WEBHOOK` secret in your repo | [Create webhook](https://api.slack.com/messaging/webhooks)

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
      # Optional: Slack notifications via OAuth (recommended)
      slack_channel_id: 'C01234567'  # Your Slack channel ID
      slack_user_id: 'U01234567'     # User to mention (optional)
    secrets:
      DOCSYNC_GITHUB_TOKEN: ${{ secrets.DOCSYNC_GITHUB_TOKEN }}
      DOCSYNC_ANTHROPIC_API_KEY: ${{ secrets.DOCSYNC_ANTHROPIC_API_KEY }}
      # Optional: Legacy webhook support
      # DOCSYNC_SLACK_WEBHOOK: ${{ secrets.DOCSYNC_SLACK_WEBHOOK }}

  docsync-on-comment:
    if: |
      github.event_name == 'issue_comment' &&
      github.event.issue.pull_request != null &&
      (startsWith(github.event.comment.body, '@docbot') || startsWith(github.event.comment.body, 'docsync:'))
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
@docbot approve          # Approve proposed changes
@docbot reject           # Reject proposed changes
@docbot revert last change
```

**Legacy format (still supported):**
```
docsync: Add installation instructions for Windows users
```

**Features:**
- ✨ **Structured commands** - Clear, predictable interface (`@docbot <action> <target>`)
- 🎯 **Specific actions** - Update, clarify, add examples, expand, fix, or revert
- 👀 **Approval workflow** - Review changes before they're committed with `@docbot approve/reject`
- 🔄 **Revert capability** - Undo changes with `@docbot revert last change`
- 📋 **Auto-guide** - Each DocSync PR includes a comment with all available commands
- 🤖 **AI-powered** - Claude analyzes context and applies changes intelligently

**Important:**
- Commands MUST start with `@docbot` (case-insensitive)
- Only works on DocSync-generated PRs (with `docsync-ai` or `automated` labels)
- AI makes the final decision - may reject, modify, or improve suggestions
- Provides quality control and sanitization of user input
- Changes require explicit approval before being committed (safer workflow)

### How It Works

1. **Merge Trigger**: When PR merges → AI analyzes diff → Creates doc PR if significant changes detected → Posts @docbot usage guide
2. **Comment Trigger**: When user comments `@docbot <command>` → AI parses command → Generates changes → Shows diff for review
3. **Approval Workflow**: User reviews diff → Comments `@docbot approve` to apply or `@docbot reject` to discard → Changes are only committed after approval
4. **AI Decision Making**: Claude AI evaluates all suggestions and makes expert decisions on documentation quality
5. **Safety**: Validates PRs, sanitizes input, only modifies documentation files, requires approval before committing
6. **Revert Support**: Can undo changes with `@docbot revert last change` for easy rollback

See [example-docsync-workflow.yml](example-docsync-workflow.yml) for a complete configuration example with detailed comments.

---

## 🔔 Slack Notifications Setup Guide

### Creating a Slack App for OAuth Token

If you want to use the OAuth token method (recommended), follow these steps:

#### 1. **Create Slack App**
1. Go to [Slack API](https://api.slack.com/apps)
2. Click **"Create New App"** → **"From scratch"**
3. Name your app: "DocSync AI" (or any name)
4. Select your workspace

#### 2. **Configure OAuth & Permissions**
1. In the left sidebar, click **"OAuth & Permissions"**
2. Scroll to **"Scopes"** section
3. Add **Bot Token Scopes**:
   - `chat:write` - Post messages to channels
   - `chat:write.public` - (Optional) Post to public channels without joining

#### 3. **Install App to Workspace**
1. Scroll to top of **"OAuth & Permissions"** page
2. Click **"Install to Workspace"**
3. Review permissions and click **"Allow"**
4. Copy the **"Bot User OAuth Token"** (starts with `xoxb-`)
   - Store this as `SLACK_OAUTH_TOKEN` in **shared-actions repo secrets**

#### 4. **Get Channel ID**
1. Open Slack desktop/web app
2. Right-click on the channel where you want notifications
3. Select **"View channel details"**
4. Scroll down and copy the **Channel ID** (starts with `C`)
   - Example: `C01234ABCDE`
   - Use this as `slack_channel_id` input in workflow

#### 5. **Get User ID (Optional)**
1. Click on your profile picture in Slack
2. Select **"Profile"**
3. Click **"More"** (three dots)
4. Select **"Copy member ID"** (starts with `U`)
   - Example: `U01234ABCDE`
   - Use this as `slack_user_id` input in workflow

#### 6. **Add Bot to Channel**
1. Open the Slack channel
2. Type `/invite @DocSync AI` (or your app name)
3. The bot must be in the channel to post messages

### Configuration Example

**In shared-actions repo:**
```
Settings > Secrets and variables > Actions > Repository secrets
Add: SLACK_OAUTH_TOKEN = xoxb-your-token-here
```

**In consuming repo workflow:**
```yaml
with:
  slack_channel_id: 'C01234ABCDE'  # Required
  slack_user_id: 'U01234ABCDE'     # Optional (for mentions)
```

### Comparison: OAuth vs Webhook

| Feature | OAuth Token (Recommended) | Webhook (Legacy) |
|---------|---------------------------|------------------|
| **Setup** | One-time in shared-actions | Per repository |
| **Security** | Token stored centrally | URL per repo |
| **Flexibility** | Multiple channels, user mentions | Single channel |
| **Maintenance** | Easy to rotate token | Need to update all repos |
| **User Mentions** | ✅ Yes (with User ID) | ⚠️ Limited |

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
