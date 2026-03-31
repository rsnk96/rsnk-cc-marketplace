---
description: Install the Claude Code status line shell script to ~/.claude/scripts/statusline.sh
argument-hint: "[--force] [--no-quotes]"
allowed-tools: [Bash(mkdir:*), Bash(tee:*), Bash(chmod:*), Bash(which:*), Bash(echo:*), Bash(test:*), Bash(cat:*), Bash(jq:*), Bash(mv:*)]
---

# Install Statusline

This command writes a shell script to `~/.claude/scripts/statusline.sh` that renders a rich status line for Claude Code sessions (branch, model, cost, duration, lines changed, token usage, and optionally a quote).

## Behavior

- If the target file exists and `--force` is not provided, confirm before overwriting.
- Ensures the `~/.claude/scripts` directory exists and sets the script executable.
- Verifies `jq` is available (required by the script) and warns if missing.
- Automatically configures `~/.claude/settings.json` to enable the status line.
- Supports `--no-quotes` to hide the quote from the status line.
- If `statusLine` is already configured and `--force` is not provided, skips configuration update.
- Validates the installed script by running it with sample JSON after installation.

## Steps

1) Check for `jq` and warn if not present (continue anyway):

- Run: `which jq || echo "Warning: jq not found. Please install jq to enable the status line parser."`

2) Create the scripts directory:

- Run: `mkdir -p ~/.claude/scripts`

3) If the file exists and `--force` is not passed, prompt for confirmation. Otherwise, proceed to write the file using a here-doc with `tee` and then mark it executable.

- Run:

```
test -f ~/.claude/scripts/statusline.sh \
  && ! echo "$ARGUMENTS" | grep -q -- '--force' \
  && echo "~/.claude/scripts/statusline.sh already exists. Re-run with --force to overwrite." \
  && exit 0 || true
tee ~/.claude/scripts/statusline.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

# Status line script for Claude Code
# Displays: Branch, Model, Cost, Duration, Lines changed, Token usage, and an optional quote

# Parse flags
NO_QUOTES=0
for arg in "$@"; do
  case "$arg" in
    --no-quotes)
      NO_QUOTES=1
      ;;
  esac
done

# Also allow env toggle for flexibility
if [ "${CLAUDE_STATUSLINE_NO_QUOTES:-}" = "1" ]; then
  NO_QUOTES=1
fi

# Check if jq is installed
if ! command -v jq >/dev/null 2>&1; then
  echo "Error: jq is required but not installed. Please install jq first." >&2
  exit 1
fi

# Read JSON input from stdin
input=$(cat)

# Validate JSON input
if [ -z "$input" ]; then
  echo "Error: No JSON input received from Claude Code." >&2
  exit 1
fi

# Extract data from JSON
cost=$(echo "$input" | jq -r '.cost.total_cost_usd // 0' 2>/dev/null)
duration_ms=$(echo "$input" | jq -r '.cost.total_duration_ms // 0' 2>/dev/null)
lines_added=$(echo "$input" | jq -r '.cost.total_lines_added // 0' 2>/dev/null)
lines_removed=$(echo "$input" | jq -r '.cost.total_lines_removed // 0' 2>/dev/null)
model_name=$(echo "$input" | jq -r '.model.display_name // "Sonnet 4.5"' 2>/dev/null)
model_id=$(echo "$input" | jq -r '.model.id // ""' 2>/dev/null)
workspace_dir=$(echo "$input" | jq -r '.workspace.current_dir // .workspace.project_dir // ""' 2>/dev/null)
ctx_used_pct=$(echo "$input" | jq -r '.context_window.used_percentage // 0' 2>/dev/null)
ctx_window_size=$(echo "$input" | jq -r '.context_window.context_window_size // 200000' 2>/dev/null)

# Validate critical values were parsed successfully
if [ -z "$cost" ] || [ "$cost" = "null" ]; then
  echo "Error: Failed to parse session data. Invalid or malformed JSON input." >&2
  exit 1
fi

# Validate numeric fields to prevent arithmetic injection
[[ "$duration_ms" =~ ^[0-9]+$ ]] || duration_ms=0
[[ "$lines_added" =~ ^[0-9]+$ ]] || lines_added=0
[[ "$lines_removed" =~ ^[0-9]+$ ]] || lines_removed=0
[[ "$ctx_used_pct" =~ ^[0-9]+$ ]] || ctx_used_pct=0
[[ "$ctx_window_size" =~ ^[0-9]+$ ]] || ctx_window_size=200000

# Get current git branch (use workspace directory if available)
if [ -n "$workspace_dir" ] && [[ "$workspace_dir" =~ ^/ ]] && [ -d "$workspace_dir" ]; then
  branch=$(git -C "$workspace_dir" rev-parse --abbrev-ref HEAD 2>/dev/null || echo "no-branch")
else
  branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "no-branch")
fi

# Format duration (convert milliseconds to seconds)
if [ "$duration_ms" != "0" ] && [ "$duration_ms" != "null" ]; then
  duration_seconds=$((duration_ms / 1000))
  minutes=$((duration_seconds / 60))
  seconds=$((duration_seconds % 60))
  if [ $minutes -gt 0 ]; then
    duration_str="${minutes}m${seconds}s"
  else
    duration_str="${seconds}s"
  fi
else
  duration_str="0s"
fi

# Format token usage: derive context fill from window size × used percentage
ctx_tokens=$(( ctx_window_size * ctx_used_pct / 100 ))
if [ "$ctx_tokens" -ge 1000 ]; then
  token_str="$((ctx_tokens / 1000))k tok"
else
  token_str="${ctx_tokens} tok"
fi
token_display="${token_str} · ${ctx_used_pct}% ctx"

if [ "$NO_QUOTES" -ne 1 ]; then
  # Fetch quote with 5-minute caching
  get_quote() {
    local cache_file="/tmp/claude_statusline_quote_$(id -u).txt"
    local cache_max_age=300  # 5 minutes in seconds
    local fallback_quote="\"Code is poetry\" - Anonymous"

    # Check if cache exists and is fresh (less than 5 minutes old)
    if [ -f "$cache_file" ]; then
      # Portable mtime check: GNU `stat -c %Y` first, then BSD `stat -f %m`.
      # Avoid inline arithmetic with command substitution to prevent multi-line issues.
      local now mtime cache_age
      now=$(date +%s)
      if mtime=$(stat -c %Y "$cache_file" 2>/dev/null); then
        :
      else
        mtime=$(stat -f %m "$cache_file" 2>/dev/null || echo 0)
      fi
      # Ensure numeric value (defensive in case of unexpected output)
      mtime=${mtime//[^0-9]/}
      [ -z "$mtime" ] && mtime=0
      cache_age=$(( now - mtime ))

      # Use cached quote if it's less than 5 minutes old
      if [ "$cache_age" -lt "$cache_max_age" ]; then
        cat "$cache_file"
        return 0
      fi
    fi

    # Cache is stale or doesn't exist - try to fetch new quote from API
    local api_response
    api_response=$(curl -s --max-time 2 "https://zenquotes.io/api/random" 2>/dev/null)
    local curl_exit_code=$?

    if [ $curl_exit_code -eq 0 ] && [ -n "$api_response" ]; then
      # Parse the JSON response (ZenQuotes returns an array, so use .[0])
      local quote_text quote_author
      quote_text=$(echo "$api_response" | jq -r '.[0].q // empty' 2>/dev/null)
      quote_author=$(echo "$api_response" | jq -r '.[0].a // "Unknown"' 2>/dev/null)

      if [ -n "$quote_text" ] && [ "$quote_text" != "null" ] && [ "$quote_text" != "empty" ]; then
        # Format quote and save to cache
        local formatted_quote="\"$quote_text\" - $quote_author"
        echo "$formatted_quote" > "$cache_file"
        echo "$formatted_quote"
        return 0
      fi
    fi

    # API failed - try to use old cached quote as fallback
    if [ -f "$cache_file" ]; then
      cat "$cache_file"
      return 0
    fi

    # No cache available, use fallback
    echo "$fallback_quote"
  }

  quote=$(get_quote)
fi

# Prepare model display string (include ID only if available)
if [ -n "$model_id" ] && [ "$model_id" != "null" ]; then
  model_display="$model_name \033[90m($model_id)\033[0m"
else
  model_display="$model_name"
fi

# Apply dim gray color to each word in the quote to ensure color persists on line wrap
# This workaround applies the color code to each word individually
apply_color_per_word() {
  local text="$1"
  local color_code="\033[90m"
  local reset_code="\033[0m"

  # Split the text into words and apply color to each word
  local -a words
  read -ra words <<<"$text"
  local result=""
  local word

  for word in "${words[@]}"; do
    result+="${color_code}${word}${reset_code} "
  done

  # Remove trailing space and output
  echo -n "${result% }"
}

if [ "$NO_QUOTES" -ne 1 ]; then
  colored_quote=$(apply_color_per_word "$quote")
  # Build and print status line with emojis, colors, and quote
  printf "🌿 \033[32m%s\033[0m | 🤖 \033[36m%b\033[0m | 💰 \033[33m\$%.4f\033[0m | ⏱️ \033[37m%s\033[0m | 📝 \033[32m+%s\033[0m/\033[31m-%s\033[0m | 🧠 \033[35m%s\033[0m | 💬 %b" \
    "$branch" \
    "$model_display" \
    "$cost" \
    "$duration_str" \
    "$lines_added" \
    "$lines_removed" \
    "$token_display" \
    "$colored_quote"
else
  # Print without quote segment
  printf "🌿 \033[32m%s\033[0m | 🤖 \033[36m%b\033[0m | 💰 \033[33m\$%.4f\033[0m | ⏱️ \033[37m%s\033[0m | 📝 \033[32m+%s\033[0m/\033[31m-%s\033[0m | 🧠 \033[35m%s\033[0m" \
    "$branch" \
    "$model_display" \
    "$cost" \
    "$duration_str" \
    "$lines_added" \
    "$lines_removed" \
    "$token_display"
fi
EOF
chmod +x ~/.claude/scripts/statusline.sh
```

4) Print success message:

- Run: `echo "Installed: ~/.claude/scripts/statusline.sh"`

5) Configure `~/.claude/settings.json` to enable the status line:

- First, check if `~/.claude/settings.json` exists. If not, create it with minimal JSON structure:

```bash
if [ ! -f ~/.claude/settings.json ]; then
  echo '{"$schema":"https://json.schemastore.org/claude-code-settings.json"}' > ~/.claude/settings.json
fi
```

- Check if `statusLine` is already configured (skip if already set and `--force` is not provided):

```bash
# Ensure jq is available
if ! command -v jq >/dev/null 2>&1; then
  echo "Error: jq is required but not installed." >&2
  exit 1
fi

# Validate JSON structure before reading fields
if ! jq empty ~/.claude/settings.json >/dev/null 2>&1; then
  echo "Error: ~/.claude/settings.json contains invalid JSON. Please fix or remove the file." >&2
  exit 1
fi

# Then check for existing statusLine configuration
if jq -e '.statusLine' ~/.claude/settings.json >/dev/null 2>&1 \
   && ! echo "$ARGUMENTS" | grep -q -- '--force'; then
  echo "statusLine already configured in ~/.claude/settings.json (use --force to overwrite)"
else
  # Add or update statusLine configuration with backup + error handling
  cp -p ~/.claude/settings.json ~/.claude/settings.json.backup
  tmp_file=$(mktemp ~/.claude/settings.json.tmp.XXXXXX)
  trap 'rm -f "$tmp_file"' EXIT

  # Prepare command, optionally including --no-quotes
  status_cmd="bash ~/.claude/scripts/statusline.sh"
  if echo "$ARGUMENTS" | grep -q -- '--no-quotes'; then
    status_cmd="$status_cmd --no-quotes"
  fi

  if ! jq --arg cmd "$status_cmd" '.statusLine = {"type": "command", "command": $cmd}' \
       ~/.claude/settings.json > "$tmp_file"; then
    echo "Error: Failed to update settings.json. Backup preserved at ~/.claude/settings.json.backup" >&2
    rm -f "$tmp_file"
    exit 1
  fi

  # Verify the generated file is not empty and contains valid JSON
  if [ ! -s "$tmp_file" ]; then
    echo "Error: Generated settings.json is empty. Backup preserved at ~/.claude/settings.json.backup" >&2
    rm -f "$tmp_file"
    exit 1
  fi

  if ! jq empty "$tmp_file" >/dev/null 2>&1; then
    echo "Error: Generated settings.json contains invalid JSON. Backup preserved at ~/.claude/settings.json.backup" >&2
    rm -f "$tmp_file"
    exit 1
  fi

  mv "$tmp_file" ~/.claude/settings.json
  trap - EXIT
  echo "Configured statusLine in ~/.claude/settings.json (backup at ~/.claude/settings.json.backup)"
fi
```

6) Validate the installed script by running it with sample JSON:

- Run:

```bash
validation_output=$(echo '{"cost":{"total_cost_usd":0.0123,"total_duration_ms":6543,"total_lines_added":10,"total_lines_removed":2},"model":{"display_name":"Sonnet 4.6","id":"claude-sonnet-4-6"},"workspace":{"project_dir":"/tmp"},"context_window":{"total_input_tokens":15234,"total_output_tokens":4521,"used_percentage":8}}' \
  | ~/.claude/scripts/statusline.sh --no-quotes 2>&1)
exit_code=$?

if [ $exit_code -ne 0 ]; then
  echo "✗ Script validation failed (exit $exit_code): $validation_output" >&2
  exit 1
fi

# Check that expected segments appear in the output
missing=""
echo "$validation_output" | grep -q '\$0.0123' || missing="$missing cost"
echo "$validation_output" | grep -q '+10'       || missing="$missing lines_added"
echo "$validation_output" | grep -q 'tok'       || missing="$missing tokens"
echo "$validation_output" | grep -q '8% ctx'    || missing="$missing ctx_pct"

if [ -n "$missing" ]; then
  echo "✗ Script validation failed — missing fields in output:$missing" >&2
  echo "  Output was: $validation_output" >&2
  exit 1
fi

echo "✓ Script validated successfully"
echo "  Preview: $validation_output"
```

## Notes

- The script expects JSON input from Claude Code and requires `jq` at runtime. Quotes are fetched via `curl` with a 5-minute cache (user-specific path `/tmp/claude_statusline_quote_<uid>.txt`) and a graceful offline fallback. Use `--no-quotes` (or set env `CLAUDE_STATUSLINE_NO_QUOTES=1`) to hide quotes entirely.
- Token display shows cumulative session tokens and context window usage percentage: e.g. `15k tok · 8% ctx`.
- The command automatically updates `~/.claude/settings.json` to enable the status line. Use `--force` to overwrite existing configurations.
- A backup of the previous settings is saved to `~/.claude/settings.json.backup`. Restore with `mv ~/.claude/settings.json.backup ~/.claude/settings.json` if needed.
- To test locally, see the separate "Preview Statusline" command.
