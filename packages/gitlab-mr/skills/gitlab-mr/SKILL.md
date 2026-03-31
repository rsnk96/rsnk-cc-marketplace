---
name: gitlab-mr
description: Creates GitLab Merge Requests using the glab CLI with gitmoji-prefixed titles and a structured MR body. Use when the user asks to create a GitLab merge request or MR.
license: MIT
allowed-tools: Read, Grep, Glob, Bash, Agent, Shell(git status), Shell(git diff), Shell(git log), Shell(git branch), Shell(git remote), Shell(git rev-parse), Shell(git push), Shell(git checkout), Shell(git add), Shell(git commit), Shell(glab mr create:*), Shell(glab mr view:*), Shell(glab auth status)
metadata:
  version: "1.0.0"
  domain: git
  triggers: merge request, MR, create MR, open MR, gitlab
---

# GitLab Merge Request Creator

## Orchestration Workflow

When invoked, follow these steps exactly:

### Step 0 — Verify GitLab remote

Run:
```bash
git remote -v
```

Check that the remote URL (origin or upstream) contains `gitlab` (e.g., `gitlab.com`, or a self-hosted GitLab instance). If the remote is NOT a GitLab remote (e.g., it's GitHub, Bitbucket, etc.), **stop immediately** and tell the user this skill is for GitLab only, and suggest using `gh` for GitHub instead.

Also verify `glab` is available:
```bash
command -v glab
```

If `glab` is not installed, stop and inform the user they need to install it (`https://gitlab.com/gitlab-org/cli`).

### Step 1 — Gather context

Run the following commands in parallel to understand the current state:

```bash
git status
git branch --show-current
git log --oneline -20
git diff --stat
git diff HEAD
```

From this, determine:
- The current branch name
- Whether there are uncommitted changes (if so, ask the user if they want to commit first)
- The full set of changes that will be part of the MR (all commits diverged from the target branch)

If on `main` or `master`, ask the user to create or switch to a feature branch first.

Determine the target/base branch by checking which of `main`, `master`, or `develop` exists:
```bash
git branch --list main master develop
```

Then get the full diff against the target branch:
```bash
git log --oneline <target_branch>...HEAD
git diff <target_branch>...HEAD
```

### Step 2 — Run code review before creating the MR

Before composing the MR, invoke the `code-reviewer` skill to review all changes on this branch. This catches bugs, security issues, and code smells before they reach the MR.

- If the review finds **critical or major issues**, fix them and commit the fixes before proceeding.
- If the review finds only **minor issues**, note them but proceed with MR creation. Mention unresolved minor findings in the MR description if relevant.
- If the review finds **no issues**, proceed directly.

### Step 3 — Choose a gitmoji and craft the title

Based on the diff and commit history, select the most appropriate gitmoji for the MR title. Use standard gitmojis:

| Emoji | Code | Use when |
|-------|------|----------|
| ✨ | `:sparkles:` | Introducing a new feature |
| 🐛 | `:bug:` | Fixing a bug |
| 🔥 | `:fire:` | Removing code or files |
| 📝 | `:memo:` | Adding or updating documentation |
| 🚀 | `:rocket:` | Deploying stuff / performance improvements |
| 💄 | `:lipstick:` | Updating UI and style files |
| 🎉 | `:tada:` | Initial commit / beginning a project |
| ✅ | `:white_check_mark:` | Adding or updating tests |
| 🔒 | `:lock:` | Fixing security issues |
| 🔧 | `:wrench:` | Changing configuration files |
| ♻️ | `:recycle:` | Refactoring code |
| ⬆️ | `:arrow_up:` | Upgrading dependencies |
| ⬇️ | `:arrow_down:` | Downgrading dependencies |
| 🏗️ | `:building_construction:` | Making architectural changes |
| 🚚 | `:truck:` | Moving or renaming files/resources |
| 💚 | `:green_heart:` | Fixing CI build |
| 👷 | `:construction_worker:` | Adding or updating CI build system |
| 🗃️ | `:card_file_box:` | Database related changes |
| 🔀 | `:twisted_rightwards_arrows:` | Merging branches |
| ⚡ | `:zap:` | Improving performance |
| 🩹 | `:adhesive_bandage:` | Simple fix for a non-critical issue |
| 🧹 | `:broom:` | Cleanup / chores |

Use the actual Unicode emoji character (not the `:code:` shorthand) at the start of the title. Follow it with a concise, descriptive title in imperative mood (e.g., "✨ Add user authentication endpoint"). Keep the title under 72 characters.

### Step 4 — Compose the MR body

Construct the MR body using this template. Fill in each section thoughtfully based on the diff analysis. Do not line break unnecessarily — there is no limit on line length.

```
<!-- Provide a general summary of your changes in the Title above -->

### Motivation and context
<!-- Why is this change required? What problem does it solve? If it fixes an open issue, please link to the issue here from Lists/Jira/Gitlab. Describe your changes in detail, add screenshots. -->

<Write a clear, detailed explanation of WHY this change is being made. Reference any related issues, tickets, or discussions. Describe the problem being solved or the feature being added. Include specific details about what changed and why those changes were necessary.>

### How has this been tested?
<!-- Please describe in detail how you tested your changes. Include details of your testing environment, and the tests you ran to see how your change affects other areas of the code, etc. -->

<Describe testing approach. If there are automated tests, mention them. If manual testing was done, describe the steps. If no testing was done, state that clearly.>
```

Additionally, include any of these optional sections if relevant:

- **Breaking changes** — if the MR introduces breaking changes, add a `### Breaking changes` section describing what breaks and migration steps.
- **Screenshots / recordings** — if the MR involves UI changes, add a `### Screenshots` section (leave a placeholder for the user to add them).
- **Related issues** — if you can detect issue references from branch name or commit messages (e.g., `#123`, `PROJ-456`), add a `### Related issues` section with `Closes #123` or `Relates to #123`.
- **Checklist** — if the project has a contributing guide or MR template, incorporate its checklist.
- **Performance impact** — if the changes could affect performance, add a `### Performance impact` section.
- **Dependencies** — if new dependencies were added or updated, add a `### Dependencies` section listing them.
- **CI pipeline impact** — if the MR changes CI configuration, test infrastructure, Docker setup, or code paths exercised by specific CI stages (validate, test, build, push), add a `### CI pipeline impact` section noting which stages may be affected and why.
- **Configuration changes** — if the MR modifies or adds TOML/YAML/JSON configuration files or changes config parsing logic, add a `### Configuration changes` section listing the config keys added/modified/removed and their default values.
- **Deployment notes** — if the MR requires container rebuilds, service restarts, migration steps, or environment variable changes for deployment, add a `### Deployment notes` section with actionable steps.

### Step 5 — Push and create the MR

First, ensure the branch is pushed to the remote:
```bash
git push -u origin <current_branch>
```

Then create the MR using `glab`. Use a HEREDOC to pass the body to ensure correct formatting:

```bash
glab mr create --title "<gitmoji> <title>" --description "$(cat <<'EOF'
<MR body content>
EOF
)" --target-branch <target_branch>
```

If the user specified any additional flags (e.g., `--assignee`, `--reviewer`, `--label`, `--milestone`, `--draft`), include them.

If the changes map to common categories, suggest labels:
- Code changes to tests → `testing`
- Configuration changes → `config`
- CI changes → `ci-cd`
- Documentation → `docs`
- Performance work → `performance`

Include `--label` flags if the user specified them or if they can be inferred from the change type. Include `--milestone` if inferable from the branch name (e.g., branch `release/v2.1` → milestone `v2.1`).

### Step 6 — Report back

After creating the MR, display:
- The MR URL
- The title used
- A brief summary of what was included

---

## Constraints

**Do:**
- Always verify the remote is GitLab before proceeding
- Always use the Unicode emoji character in the title, not the `:shortcode:`
- Write detailed, meaningful MR descriptions — not boilerplate
- Reference issue numbers if detectable from branch names or commits
- Ask the user before committing if there are uncommitted changes

**Don't:**
- Don't create the MR if there are no changes to merge
- Don't force-push without explicit user permission
- Don't use this skill for non-GitLab remotes
- Don't create MRs targeting the same branch you're on
- Don't guess issue numbers — only reference them if clearly present in branch name or commit messages
