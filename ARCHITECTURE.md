# 🏗️ DocSync AI - Architecture

## Overview

DocSync AI uses a **reusable workflow** architecture that allows multiple repositories to use the same shared documentation automation logic.

---

## 📁 Repository Structure

```
shared-actions/ (this repository)
├── .github/
│   ├── workflows/
│   │   ├── docsync-ai.yml           ← Reusable workflow for PR merges
│   │   └── docsync-ai-refine.yml    ← Reusable workflow for comments
│   └── actions/
│       ├── docsync_ai/
│       │   └── action.yml           ← Main documentation sync logic
│       └── docsync_ai_refine/
│           └── action.yml           ← Refinement logic
└── docs/
    ├── DOCSYNC_AI_USAGE.md          ← Complete user guide
    ├── WORKFLOW_TEMPLATE.yml         ← Copy-paste template
    └── ARCHITECTURE.md               ← This file

user-repository/ (any other repository)
└── .github/
    └── workflows/
        └── docsync-ai.yml           ← ONE file that calls BOTH workflows
```

---

## 🔄 Workflow Architecture

### Why Two Reusable Workflows?

GitHub reusable workflows (`workflow_call`) receive event context from the calling workflow. Different events have different contexts that cannot be mixed:

| Workflow | Trigger Event | Event Context |
|----------|---------------|---------------|
| `docsync-ai.yml` | `pull_request` | PR number, title, merge status, diff |
| `docsync-ai-refine.yml` | `issue_comment` | Comment body, comment ID, author |

**These contexts are incompatible**, so we need separate reusable workflows.

### User Experience: ONE File

Despite having two reusable workflows in the shared repo, **users only add ONE workflow file** to their repository:

```yaml
# user-repo/.github/workflows/docsync-ai.yml

name: 📚 DocSync AI

on:
  pull_request:      # Trigger for sync
  issue_comment:     # Trigger for refinement

jobs:
  sync-docs:
    if: pull_request event
    uses: shared-actions/.../docsync-ai.yml

  refine-docs:
    if: issue_comment event
    uses: shared-actions/.../docsync-ai-refine.yml
```

---

## 🎯 Flow Diagrams

### Flow 1: Automatic Documentation Sync

```
Developer                 GitHub                    DocSync AI                 Claude API
    |                        |                           |                          |
    |--[Merge PR to master]->|                           |                          |
    |                        |                           |                          |
    |                        |--[Trigger workflow]------>|                          |
    |                        |                           |                          |
    |                        |                           |--[Check PR title]        |
    |                        |                           |  (Skip if DocSync PR)    |
    |                        |                           |                          |
    |                        |                           |--[Get PR diff]---------->|
    |                        |                           |                          |
    |                        |                           |<-[Analyze changes]-------|
    |                        |                           |  (Significance check)    |
    |                        |                           |                          |
    |                        |                           |--[Update README.md]      |
    |                        |                           |                          |
    |                        |<--[Create doc PR]---------|
    |                        |                           |
    |<-[Slack notification]--|                           |
    |  "Doc PR ready"        |                           |
```

### Flow 2: Human Feedback Refinement

```
Developer                 GitHub                    DocSync AI                 Claude API
    |                        |                           |                          |
    |--[@docsync comment]--->|                           |                          |
    |  "Add more examples"   |                           |                          |
    |                        |--[Trigger workflow]------>|                          |
    |                        |                           |                          |
    |                        |                           |--[Sanitize comment]      |
    |                        |                           |  (Security check)        |
    |                        |                           |                          |
    |                        |                           |--[Get original PR]       |
    |                        |                           |  (Context gathering)     |
    |                        |                           |                          |
    |                        |                           |--[Refine docs]---------->|
    |                        |                           |                          |
    |                        |                           |<-[Refined content]-------|
    |                        |                           |                          |
    |                        |                           |--[Commit to PR branch]   |
    |                        |                           |                          |
    |                        |<--[Reply to comment]------|
    |<-[GitHub notification]-|   "✅ Updated!"           |
```

---

## 🔒 Security Architecture

### Input Sanitization (Refinement Feature)

```
User Comment
     |
     v
[Extract @docsync command]
     |
     v
[Blocked Pattern Check]
     |-- secret, token, password → ❌ BLOCKED
     |-- .env, api_key, credential → ❌ BLOCKED
     |-- .yml, .json, package.json → ❌ BLOCKED
     |-- workflow, action, sql → ❌ BLOCKED
     |
     v
[Length Validation]
     |-- < 10 chars → ❌ REJECTED
     |
     v
[Context Validation]
     |-- Not a DocSync PR → ❌ SKIPPED
     |
     v
✅ SAFE - Process Refinement
```

### File Modification Control

Both workflows enforce strict file modification rules:

```
Changed Files Check
     |
     v
[Get git diff]
     |
     v
[For each changed file]
     |
     |-- README.md → ✅ ALLOWED
     |-- CLAUDE.md → ✅ ALLOWED
     |-- *.js, *.yml, *.json → ❌ BLOCKED (exit with error)
     |-- .env, secrets → ❌ BLOCKED (exit with error)
     |
     v
✅ Safe to create PR
```

---

## 🔄 Concurrency Control

### Main Workflow
```yaml
concurrency:
  group: docsync-ai-${{ github.repository }}
  cancel-in-progress: false
```
- One sync per repository at a time
- Queues additional triggers

### Refinement Workflow
```yaml
concurrency:
  group: docsync-ai-refine-${{ github.repository }}-${{ github.event.issue.number }}
  cancel-in-progress: false
```
- One refinement per PR at a time
- Allows concurrent refinements on different PRs

---

## 📊 Component Responsibilities

### Reusable Workflows (`.github/workflows/`)
- **Purpose**: Entry points for different GitHub events
- **Responsibilities**:
  - Validate event context
  - Configure job permissions
  - Call composite actions
  - Handle notifications
  - Provide debug information

### Composite Actions (`.github/actions/`)
- **Purpose**: Core business logic
- **Responsibilities**:
  - Interact with GitHub API
  - Call Claude API
  - Process files
  - Validate changes
  - Create commits/PRs

### Why This Split?

| Layer | Can Be | Cannot Be |
|-------|--------|-----------|
| Reusable Workflow | Called from other repos | Nested (workflow calls workflow) |
| Composite Action | Called from workflows | Called from other actions |

This two-layer architecture provides:
- ✅ Clean separation of concerns
- ✅ Reusability across repositories
- ✅ Easier testing and maintenance
- ✅ Clear responsibility boundaries

---

## 🚀 Deployment Architecture

### Shared Actions Repository (Central)
```
master branch
    |
    |-- Used by all repositories
    |-- Single source of truth
    |-- No per-repo configuration needed
```

### User Repositories (Distributed)
```
Each repository:
    |-- Adds ONE workflow file
    |-- Configures THREE secrets
    |-- Triggers automatically
    |-- No maintenance required
```

### Benefits
- ✅ Central updates benefit all repos
- ✅ Consistent behavior across projects
- ✅ Easy version management (@master or @v1.0.0)
- ✅ Minimal per-repo configuration

---

## 📈 Scaling Considerations

### API Usage
- **Claude API**: ~1 call per PR merge (only if significant changes)
- **GitHub API**: Multiple calls per workflow run
- **Rate Limits**: Handled by GitHub token permissions

### Cost Management
- **Re-trigger Prevention**: Skips doc PR merges (saves credits)
- **Significance Check**: Only updates for meaningful changes
- **Efficient Context**: Only sends necessary code to Claude

### Performance
- **Checkout**: Uses `fetch-depth: 0` only when needed
- **Caching**: No caching needed (stateless workflows)
- **Concurrency**: Controlled to prevent conflicts

---

## 🔧 Extension Points

Want to customize DocSync AI? Here are the extension points:

### 1. Custom Prompts
Edit the prompt in `docsync_ai/action.yml` or `docsync_ai_refine/action.yml`

### 2. Additional Validations
Add checks in the sanitization step of `docsync_ai_refine/action.yml`

### 3. Different Claude Models
Change the model in the API call (currently `claude-3-7-sonnet-latest`)

### 4. Additional Notifications
Add steps to the reusable workflows for other services (email, Teams, etc.)

### 5. Custom Labels
Pass different labels via the `pr_labels` input parameter

---

## 🎯 Design Principles

1. **Security First**: Multiple layers of input validation
2. **User Safety**: Never modify files other than documentation
3. **Simplicity**: One file for users to add
4. **Maintainability**: Centralized logic, distributed execution
5. **Transparency**: Comprehensive logging and debugging
6. **Cost Conscious**: Avoid unnecessary API calls
7. **Human-in-the-Loop**: AI assists, humans approve

---

## 📚 Related Documents

- [DOCSYNC_AI_USAGE.md](./DOCSYNC_AI_USAGE.md) - Complete user guide
- [WORKFLOW_TEMPLATE.yml](./WORKFLOW_TEMPLATE.yml) - Copy-paste template
- [README.md](./README.md) - Repository overview

---

## ❓ FAQ

**Q: Why not merge both workflows into one?**
A: Reusable workflows with `workflow_call` can only handle one event type. Different events (pull_request vs issue_comment) have incompatible contexts.

**Q: Can users customize the workflows?**
A: Yes! Fork the shared-actions repo, modify the workflows, and update the `uses:` reference in your workflow file.

**Q: How do updates work?**
A: Users reference `@master` to always get the latest, or pin to a specific version like `@v1.0.0` for stability.

**Q: Can I use this with GitHub Enterprise?**
A: Yes! Works with GitHub Enterprise Server and GitHub Enterprise Cloud.

**Q: What if I have multiple documentation files?**
A: Currently supports README.md OR CLAUDE.md (whichever exists). Future versions could support multiple files.

---

**Built with ❤️ for automated documentation maintenance**
