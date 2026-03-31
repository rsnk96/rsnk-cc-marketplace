---
name: code-reviewer
description: |
  Analyzes code diffs and files to identify bugs, security vulnerabilities, code smells, and architectural concerns, then produces a structured review report with prioritized, actionable feedback. Requires the pr-review-toolkit plugin to be installed separately for comprehensive multi-agent coverage, then validates each finding through deep codebase exploration.
  TRIGGER when: user asks to "review code", "review this branch", "review changes", "check for issues", "audit code", or asks to create a PR/MR (run review BEFORE creating the MR).
  DO NOT TRIGGER when: user only asks for a quick look, explanation, or summary of changes without requesting a review.
license: MIT
# Read-only git only (no add/commit/push/checkout/reset/merge/rebase/stash apply/submodule update etc.)
allowed-tools: Read, Grep, Glob, Bash, Agent, Skill, WebSearch, Search, TaskCreate, TaskUpdate, TaskList, TaskGet, Shell(tail), Shell(xxd), Shell(python3), Shell(git show), Shell(git rev-parse), Shell(git diff), Shell(git log), Shell(git branch), Shell(git status), Shell(git blame), Shell(git grep), Shell(git ls-files), Shell(git ls-tree), Shell(git cat-file), Shell(git describe), Shell(git rev-list), Shell(git for-each-ref), Shell(git merge-base), Shell(git reflog), Shell(git worktree list), Shell(git remote), Shell(git tag), Shell(git stash list), Shell(git stash show), Shell(git config), Shell(git symbolic-ref), Shell(git name-rev), Shell(git shortlog), Shell(git count-objects), Shell(git verify-commit), Shell(git verify-tag), Shell(git submodule status), Shell(git submodule foreach)
metadata:
  version: "5.5.4"
  domain: quality
  triggers: code review, PR review, pull request, review code, code quality
---

# Code Reviewer

## Prerequisites

This skill requires the `pr-review-toolkit` plugin from the official Claude Code marketplace. Install it before using this skill:

```
/plugin install pr-review-toolkit@claude-plugins-official
```

The code-reviewer skill launches up to 9 specialized review agents:

**Required (from pr-review-toolkit):**
- `pr-review-toolkit:code-reviewer` - Code quality and bug detection
- `pr-review-toolkit:pr-test-analyzer` - Test coverage analysis
- `pr-review-toolkit:comment-analyzer` - Comment accuracy review
- `pr-review-toolkit:silent-failure-hunter` - Error handling detection
- `pr-review-toolkit:type-design-analyzer` - Type system design review
- `pr-review-toolkit:code-simplifier` - Simplification suggestions

**Always included:**
- `superpowers:code-reviewer` - Architectural and correctness review via the superpowers reviewer subagent

**Optional (if Codex is configured — run `/codex:setup` to enable):**
- `codex:review` - Codex code review pass
- `codex:adversarial-review` - Codex adversarial (red-team) review pass

Without pr-review-toolkit installed, this skill will fail at Step 2 with agent launch errors.

## Orchestration Workflow

When invoked, follow these steps exactly:

### Step 1 — Determine diff scope

**Parse user arguments to determine what to review:**

**Case 1: User provided explicit scope**
- If user provided specific commit SHA(s), branch name(s), or file path(s) as arguments, use that scope exactly.
- If user explicitly specified "unstaged", "untracked", "working tree", "uncommitted changes", or similar keywords, review unstaged/untracked changes using `git diff` and list untracked files from `git status`.

**Case 2: User said something generic like "review changes" or "review code"**
- First check what changes exist:
  ```bash
  git status --short
  ```
- If there are unstaged or untracked changes shown by `git status`, review those using `git diff` (for unstaged) and listing untracked files.
- If no unstaged/untracked changes, fall through to Case 3 (default behavior).

**Case 3: Default behavior (no explicit scope, or no unstaged changes found)**
- Run:
  ```bash
  git branch --show-current
  ```
- If the current branch is `main` or `master`: use `git diff HEAD~1` (last commit vs previous).
- If on a different branch: use `git diff main...HEAD` (all commits diverged from main). If `main` doesn’t exist, try `master`.

**Capture the scope information:**
- Store which scope was used: "unstaged/untracked changes", "last commit (HEAD vs HEAD~1)", "branch <name> vs main", or "user-specified scope: <description>"
- Capture the diff and store it
- If reviewing commits, also run `git log --oneline` for the relevant range to capture commit summary

### Step 1.5 — Detect and handle submodule changes (READ-ONLY)

Check if the diff includes any submodule reference changes (submodule pointer updates). Submodule changes appear in `git diff` as lines like `Subproject commit <sha>` or as 160000-mode entries.

```bash
git diff <same-range-as-step-1> --raw | grep '^:.*160000'
```

If any submodule references were updated:

1. **Check submodule status** to see if they are already initialized:
   ```bash
   git submodule status <path-to-submodule>
   ```

2. **For submodules that ARE already initialized** (status shows a commit SHA, not `-` prefix):
   - Capture the submodule diffs using:
     ```bash
     git diff <same-range-as-step-1> --submodule=diff -- <path-to-submodule>
     ```
   - If that fails, try:
     ```bash
     cd <path-to-submodule> && git log --oneline <old-sha>..<new-sha> && git diff <old-sha>..<new-sha> && cd -
     ```
   - Include these submodule code changes in the review with full validation

3. **For submodules that are NOT initialized** (status shows `-` prefix or error):
   - **DO NOT run `git submodule update` or any state-changing commands**
   - Note them in a "Limitations" section in the final report:
     ```
     ⚠️ **Submodule Review Limitation**: The following submodules were updated but not initialized in the working tree. Their code changes could not be reviewed:
     - path/to/submodule (old-sha → new-sha)

     To review these changes, the user should run:
     git submodule update --init <path>
     ```

4. **For initialized submodules**: Check for cross-submodule dependencies using grep (as before) to understand if they reference other submodules. If referenced submodules are also initialized, trace those calls; if not, note the limitation.

**If no submodule references were updated, skip this step entirely.**

### Step 1.75 — Check Codex availability

Before launching agents, determine whether Codex is configured and reachable:

```bash
node "/home/nikhil/.claude/plugins/marketplaces/openai-codex/plugins/codex/scripts/codex-companion.mjs" setup --json 2>/dev/null
```

- If the script **does not exist or exits non-zero**: mark Codex as **unavailable** — skip Codex agents in Step 2 and note the omission in the final report.
- If the output JSON has `auth.loggedIn` set to `false` or `auth.available` set to `false`: mark Codex as **unavailable** — tell the user to run `/codex:setup` in the final report.
- If `auth.loggedIn` is `true`: mark Codex as **available**.

Also get the git SHAs now (needed for the superpowers reviewer):
```bash
git rev-parse HEAD
git rev-parse HEAD~1   # or $(git merge-base main HEAD) if on a branch
```

### Step 2 — Launch ALL review agents IN PARALLEL

**CRITICAL: Launch all agents in a SINGLE message with multiple Agent tool calls.**

**Core 6 agents** (always launch; skip type-design-analyzer if no types changed):
- pr-review-toolkit:code-reviewer
- pr-review-toolkit:pr-test-analyzer
- pr-review-toolkit:comment-analyzer
- pr-review-toolkit:silent-failure-hunter
- pr-review-toolkit:type-design-analyzer
- pr-review-toolkit:code-simplifier — **IMPORTANT:** When invoking code-simplifier, add this instruction to the prompt: "Do NOT make any edits. Instead, provide suggestions for improvements in your report. Only analyze and suggest, never modify files."

**superpowers:code-reviewer** (always launch):

Launch a general-purpose agent with the following prompt (substitute the actual SHA values from Step 1.75 and the scope description from Step 1):

```
Invoke the superpowers:requesting-code-review skill using the Skill tool.

Context for the review:
- BASE_SHA: {BASE_SHA}
- HEAD_SHA: {HEAD_SHA}
- What was changed: {one-line summary from git log}
- Scope: {scope description from Step 1}

Follow the skill's instructions exactly to dispatch the superpowers:code-reviewer subagent
with the correct context. Return the complete reviewer output verbatim.
```

**Codex agents** (launch only if Codex is **available** per Step 1.75):

The Codex review commands have `disable-model-invocation: true` and cannot be invoked via the Skill tool. Launch two general-purpose agents that call the companion script directly via Bash.

First, map the scope from Step 1 to Codex flags:
- Branch vs main/master: `--base main --scope branch` (or `--base master --scope branch`)
- Last commit (HEAD vs HEAD~1): `--base HEAD~1 --scope branch`
- Unstaged / working tree: `--scope working-tree`

Agent A prompt (substitute `{SCOPE_FLAGS}` with the actual flags):
```
Run this Bash command:

codex exec review {SCOPE_FLAGS} --dangerously-bypass-approvals-and-sandbox

The output will be verbose. Parse it and return a structured summary containing:
- Each distinct finding with: file:line, issue description, severity estimate
- Overall verdict (approve / request changes / comment)
Do not return the raw output verbatim — condense it to actionable findings only.
If the command fails, return the error verbatim.
```

Agent B prompt (substitute `{SCOPE_FLAGS}` with the same flags):
```
Run this Bash command and return its complete stdout verbatim without summarizing or paraphrasing:

node "/home/nikhil/.claude/plugins/marketplaces/openai-codex/plugins/codex/scripts/codex-companion.mjs" adversarial-review --wait {SCOPE_FLAGS}

If the command fails or produces no output, return the error verbatim.
```

Wait for all agents to complete and capture findings from each.

If Codex agents fail (exit error, timeout, or "not configured"), capture the failure reason and note it in the final report rather than aborting the overall review.

### Step 2.5 — Enumerate ALL findings from ALL agents

**Before creating any tasks**, scan every agent's output in full and produce a complete findings table. Every distinct finding from every agent must be a row. Do not proceed to Step 3 until all agent outputs have been fully scanned.

| # | Agent | File:Line | Summary | Prelim Severity |
|---|-------|-----------|---------|-----------------|
| … | …     | …         | …       | …               |

**Rules:**
- One row per distinct finding. If two agents raise the same issue at the same location, merge into one row and note both agents.
- Do not skip findings you consider minor or obvious — list them all. Filtering happens in Step 4.
- If an agent produced no findings (e.g., codex:review sandbox failure), write "no findings / sandbox failure" in a single row so it's explicit.

After the table is complete, count the rows. The number of validation tasks created in Step 3 must equal the number of finding rows.

### Step 3 — Create validation task list

After the review-pr skill completes, create a structured task list using TaskCreate for each finding that requires validation. This helps track progress through the validation process and ensures no finding is missed.

**For each row in the findings table from Step 2.5:**

1. Create a task with:
   - **subject**: `"Validate: [file:line] - [brief issue description]"`
   - **description**: Include:
     - Full finding description from the agent
     - Which agent raised it (code-reviewer, pr-test-analyzer, comment-analyzer, silent-failure-hunter, type-design-analyzer, code-simplifier, superpowers:code-reviewer, codex:review, or codex:adversarial-review)
     - Preliminary severity estimate (Critical/Major/Minor) based on the agent's assessment
     - Validation checklist items:
       ```
       - [ ] Read full file context
       - [ ] Map dependencies (what code uses)
       - [ ] Map dependents (what uses this code)
       - [ ] Check architectural context
       - [ ] Examine tests
       - [ ] Determine: VALIDATED / FALSE POSITIVE / NEEDS CONTEXT
       - [ ] Assign final severity:
             Critical = Will break core functionality of the codebase/feature
             Major = Significant issue but doesn't break core functionality
             Minor = Low-impact issue or improvement opportunity
       ```
   - **activeForm**: `"Validating [file:line] finding"`

2. Group similar findings or findings in the same file to avoid creating too many fragmented tasks - use your judgment, but err on the side of having separate tasks for distinct concerns.

3. If there are many findings (>15), consider creating a parent task for each agent's findings, and then individual tasks as subtasks.

After creating all validation tasks, use TaskList to confirm the task list was created successfully, then proceed to Step 4.

### Step 4 — Deep validation of each finding

For each finding from any agent, regardless of which agent raised it, perform deep codebase validation. Mark each corresponding task as in_progress using TaskUpdate when you start validating it, and mark it as completed after validation.

**Validation Protocol** (do this for every single finding):

1. **Read the actual code**: Read the full file containing the finding, not just the diff hunk. Understand the complete function/class context.

2. **Map dependencies (what the code uses)**:
   - Identify all imports and function calls in the relevant code
   - Read each dependency’s implementation (1 level deep)
   - Understand what contracts and guarantees they provide
   - Check if the finding’s concern is already handled by a dependency

3. **Map dependents (what uses the code)**:
   - Grep for all references to the symbol/function/class in question
   - Read every call site across the entire codebase
   - Check if callers would be affected by the finding
   - Verify if callers provide guarantees that invalidate the finding

4. **Check architectural context**:
   - If the code is part of a protocol/IPC/shared-memory system, read both sides
   - If it touches configuration, read the config schema and validators
   - If it’s in a multi-process/threading context, understand the synchronization
   - For framework-specific patterns: understand the framework’s conventions (e.g., message buses, config systems, communication patterns)

5. **Examine tests**:
   - Find and read tests for the changed module
   - Check if tests already cover the scenario in the finding
   - Verify if test fixtures or mocks provide guarantees

6. **Verify the finding**:
   - Ask: Is this finding actually valid given the full context?
   - Is there existing handling/validation that addresses it?
   - Is the concern already prevented by design constraints?
   - Is this finding in untouched code (pre-existing, not introduced by this diff)?

Mark each finding as either:
- **[VALIDATED]** — Real issue introduced/touched by this diff, confirmed after exploration
- **[FALSE POSITIVE]** — Not actually an issue, or pre-existing issue not in scope
- **[NEEDS CONTEXT]** — Cannot definitively validate without author input

**For [VALIDATED] findings, assign severity:**
- **Critical**: Issue will break core functionality of the codebase/feature (crashes, data loss, security breach, critical functionality broken)
- **Major**: Significant issue but doesn't break core functionality (bugs, performance problems, incorrect behavior in edge cases)
- **Minor**: Low-impact issue or improvement opportunity (code quality, maintainability, minor inefficiencies)

After validating a finding, update the corresponding task using TaskUpdate:
- Set status to `completed`
- Add a comment with the validation result ([VALIDATED], [FALSE POSITIVE], or [NEEDS CONTEXT]) and final severity if validated
- Include a brief summary of why you reached this conclusion

### Step 5 — Output the final report

**Surface ONLY validated findings.** Only include findings marked as [VALIDATED] after deep codebase exploration. Do not include [FALSE POSITIVE] or [NEEDS CONTEXT] findings in the main issue sections.

**Important:** Begin with AI review disclaimer and validation statistics, then clearly state what was reviewed.

```
## 🤖 AI Code Review Report

> **Note:** This is an AI-assisted code review. All findings have been validated through deep codebase exploration, including dependency mapping, call-site analysis, and architectural context verification.
>
> **Validation Summary:** X findings identified by automated agents → Y validated as real issues → Z false positives filtered out

**Scope Reviewed:** [Clearly state what was reviewed: “branch <name> vs main (<N> commits)”, “last commit (HEAD vs HEAD~1)”, “unstaged/untracked changes”, or “user-specified: <description>”]

## Summary
[One sentence: what the PR/change does + overall verdict based on validated findings count.]

## Critical Issues (X validated)
[ONLY VALIDATED critical issues from any agent. Format: file:line — description — concrete fix — [agent-name]]
(If none: “✓ None found after deep validation.”)

## Major Issues (X validated)
[ONLY VALIDATED major/important issues from any agent. Same format.]
(If none: “✓ None found after deep validation.”)

## Minor Issues (X validated)
[ONLY VALIDATED minor issues from any agent. Same format.]
(If none: “✓ None found after deep validation.”)

## Maybe — Needs Context (X findings)
[All NEEDS CONTEXT items that require author input to validate. Format: file:line — description — what context is needed to validate — [agent-name]]
(If none: “✓ All findings were definitively validated or invalidated.”)

These findings could not be definitively classified as real issues or false positives without additional context from the author.

## Positive Feedback
[Specific patterns done well — aggregate from all agents.]

## Codex Reviews
[Include this section only if Codex agents ran. Summarize key findings from codex:review and codex:adversarial-review verbatim or as brief bullet points. If Codex was unavailable, note: “⚠️ Codex not configured — run /codex:setup to enable.”]

## Verdict: Approve / Request Changes / Comment
[Based on validated findings severity.]
```

---

## Constraints

**Do:**
- Provide specific file:line references for every issue
- Include a concrete fix or example for critical/major issues
- Praise good patterns specifically

**Don't:**
- Nitpick style when a linter is configured
- Block on personal preferences
- Accept "this is safe because of X" without independently verifying X
