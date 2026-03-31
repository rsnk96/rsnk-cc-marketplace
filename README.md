# rsnk-cc-marketplace

Personal Claude Code plugins.

## Plugins

| Plugin | Description |
|--------|-------------|
| [statusline](packages/statusline/) | Rich shell status line showing branch, model, cost, duration, diff lines, and an optional quote |
| [code-reviewer](packages/code-reviewer/) | Analyzes code diffs and files to identify bugs, security vulnerabilities, code smells, and architectural concerns. **Requires:** `pr-review-toolkit@claude-plugins-official` |
| [gitlab-mr](packages/gitlab-mr/) | Creates GitLab Merge Requests using the glab CLI with gitmoji-prefixed titles and a structured MR body |

## Install

Add the marketplace, then install the plugins you want:

```
/plugin marketplace add rsnk96/rsnk-cc-marketplace

/plugin install statusline@rsnk-cc-marketplace
/plugin install gitlab-mr@rsnk-cc-marketplace

# For code-reviewer, first install its dependency:
/plugin install pr-review-toolkit@claude-plugins-official
/plugin install code-reviewer@rsnk-cc-marketplace
```

### Keeping plugins up to date

```
/plugin update statusline
/plugin update code-reviewer
/plugin update gitlab-mr
```

## Attribution

The `statusline` plugin is adapted from [setouchi-h/cc-marketplace](https://github.com/setouchi-h/cc-marketplace) by [Kazuki Hashimoto](https://github.com/setouchi-h), licensed MIT. Colors have been adjusted for less visual intensity on dark terminals.
