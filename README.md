# Claude Code Dispatch

> One-command dispatch of development tasks to [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with automatic Telegram notification on completion. Zero polling, zero token waste.

An [OpenClaw](https://github.com/openclaw/openclaw) skill that wraps Claude Code CLI into a fire-and-forget workflow: dispatch a task, walk away, get notified when it's done.

## Features

- **Fire & Forget** — `nohup` dispatch, automatic callback via Stop Hook
- **Agent Teams** — Multi-agent parallel development with dedicated Testing Agent via structured `--agents` JSON
- **Cost Controls** — `--max-budget-usd` spend cap + `--max-turns` limit + `--fallback-model` for overload resilience
- **Git Worktree Isolation** — `--worktree` for parallel tasks in isolated branches
- **Custom Subagents** — Define specialized agents (security reviewer, testing agent, etc.) via `--agents-json`
- **Auto-Callback** — Group notifications, DM callbacks, webhook wake events
- **Rich Notifications** — Task status, duration, test results, file tree — all in one Telegram message
- **PTY Wrapper** — Reliable execution even in non-TTY environments (CI, exec, cron)
- **MCP Integration** — Load MCP servers for tasks via `--mcp-config`
- **System Prompt Customization** — `--append-system-prompt` / `--append-system-prompt-file`

## Architecture

```
dispatch.sh
  → write task-meta.json
  → launch Claude Code via claude_code_run.py (PTY)
  → [Agent Teams: --agents JSON defines Testing Agent + custom subagents]
  → Claude Code finishes → Stop/TaskCompleted hook fires automatically
    → notify-agi.sh reads meta + output
    → writes latest.json
    → sends Telegram notification
    → writes pending-wake.json (heartbeat fallback)
```

## Quick Start

### Prerequisites

- [OpenClaw](https://github.com/openclaw/openclaw) installed and configured
- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) (`claude`) installed
- A Telegram bot configured in OpenClaw (for notifications)

### Installation

Copy the skill into your OpenClaw skills directory:

```bash
cp -r claude-code-dispatch ~/.openclaw/skills/
# Or symlink
ln -s /path/to/claude-code-dispatch ~/.openclaw/skills/claude-code-dispatch
```

Set up the Stop Hook (see [Hook Setup](references/hook-setup.md)):

```bash
mkdir -p ~/.claude/hooks
cp scripts/notify-agi.sh ~/.claude/hooks/
chmod +x ~/.claude/hooks/notify-agi.sh
```

Configure hooks in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [{ "hooks": [{ "type": "command", "command": "~/.claude/hooks/notify-agi.sh" }] }],
    "TaskCompleted": [{ "hooks": [{ "type": "command", "command": "~/.claude/hooks/notify-agi.sh" }] }],
    "SessionEnd": [{ "hooks": [{ "type": "command", "command": "~/.claude/hooks/notify-agi.sh" }] }]
  }
}
```

### Usage

⚠️ **Always use `nohup` + background (`&`)** — dispatch runs until Claude Code finishes (minutes to hours).

```bash
# Simple task
nohup bash scripts/dispatch.sh \
  -p "Build a Python REST API with FastAPI" \
  -n "my-api" \
  -g "-5006066016" \
  --permission-mode bypassPermissions \
  --workdir /home/ubuntu/projects/my-api \
  > /tmp/dispatch-my-api.log 2>&1 &

# With Agent Teams (parallel dev + testing)
nohup bash scripts/dispatch.sh \
  -p "Build a Python REST API with FastAPI" \
  -n "my-api" \
  --agent-teams \
  --permission-mode bypassPermissions \
  --workdir /home/ubuntu/projects/my-api \
  > /tmp/dispatch-my-api.log 2>&1 &

# With cost controls + fallback
nohup bash scripts/dispatch.sh \
  -p "Refactor the auth module" \
  -n "auth-refactor" \
  --max-budget-usd 5.00 \
  --max-turns 50 \
  --fallback-model sonnet \
  --permission-mode bypassPermissions \
  --workdir /home/ubuntu/projects/my-app \
  > /tmp/dispatch-auth.log 2>&1 &

# With git worktree isolation
nohup bash scripts/dispatch.sh \
  -p "Implement feature X" \
  -n "feature-x" \
  --worktree feature-x \
  --permission-mode bypassPermissions \
  --workdir /home/ubuntu/projects/my-app \
  > /tmp/dispatch-feature.log 2>&1 &
```

## Parameters

| Param | Short | Required | Description |
|-------|-------|----------|-------------|
| `--prompt` | `-p` | ✅* | Task description |
| `--prompt-file` | | ✅* | Read prompt from file |
| `--name` | `-n` | | Task name for tracking |
| `--group` | `-g` | | Telegram group ID for notifications |
| `--workdir` | `-w` | | Working directory (default: cwd) |
| `--agent-teams` | | | Enable Agent Teams mode |
| `--agents-json` | | | Custom subagent definitions (JSON string) |
| `--teammate-mode` | | | Display: `auto` / `in-process` / `tmux` |
| `--permission-mode` | | | `bypassPermissions` / `plan` / `acceptEdits` / `default` |
| `--allowed-tools` | | | Tool allowlist (e.g. `"Read,Bash"`) |
| `--disallowed-tools` | | | Tool denylist |
| `--model` | | | Model override (sonnet/opus/haiku/full name) |
| `--fallback-model` | | | Fallback model when primary is overloaded |
| `--max-budget-usd` | | | Maximum dollar spend before auto-stopping |
| `--max-turns` | | | Maximum agentic turns before stopping |
| `--worktree` | | | Git worktree name for isolation |
| `--no-session-persistence` | | | Don't save session to disk |
| `--append-system-prompt` | | | Append to default system prompt |
| `--append-system-prompt-file` | | | Append system prompt from file |
| `--mcp-config` | | | MCP servers JSON file path |
| `--verbose` | | | Enable verbose logging |
| `--callback-group` | | | Telegram group for dispatching agent callback |
| `--callback-dm` | | | Telegram user ID for DM callback |
| `--callback-account` | | | Telegram bot account for DM callback |

\* One of `--prompt` or `--prompt-file` is required.

## Agent Teams

When `--agent-teams` is enabled without `--agents-json`, the dispatch script automatically defines a structured Testing Agent via the `--agents` CLI flag:

```json
{
  "testing-agent": {
    "description": "Dedicated testing agent for comprehensive test coverage",
    "prompt": "Write and run tests for all code changes...",
    "tools": ["Read", "Edit", "Write", "Bash", "Glob", "Grep"],
    "model": "sonnet"
  }
}
```

This replaces the older prompt-injection approach with Claude Code's native `--agents` flag, giving the Testing Agent its own context window, tool restrictions, and model selection.

### Custom Subagents

Pass `--agents-json` to define your own team:

```bash
--agents-json '{
  "security-reviewer": {
    "description": "Reviews code for security vulnerabilities",
    "prompt": "You are a security expert. Focus on OWASP top 10...",
    "tools": ["Read", "Grep", "Glob"],
    "model": "opus"
  },
  "perf-analyst": {
    "description": "Analyzes and optimizes performance",
    "prompt": "Profile code and suggest optimizations...",
    "tools": ["Read", "Bash", "Grep"],
    "model": "sonnet"
  }
}'
```

Each subagent is an independent Claude Code process with its own context window, sharing the same filesystem.

## Cost Controls

Control spending with:

| Flag | Description |
|------|-------------|
| `--max-budget-usd 5.00` | Hard spend cap in dollars |
| `--max-turns 50` | Maximum agentic turns |
| `--fallback-model sonnet` | Auto-switch when primary model is overloaded |

These are especially important for Agent Teams, which consume significantly more tokens.

## Git Worktree Isolation

Use `--worktree <name>` to run the task in an isolated git worktree:

```bash
--worktree feature-auth
# Claude Code runs at <repo>/.claude/worktrees/feature-auth
```

This allows parallel dispatch tasks to work on the same repo without conflicts.

## Auto-Callback Detection

If no `--callback-group` or `--callback-dm` is passed, the script looks for `dispatch-callback.json` in the working directory:

```json
// Group callback
{ "type": "group", "group": "-5189558203" }

// DM callback
{ "type": "dm", "dm": "8009709280", "account": "coding-bot" }

// Wake hook (for main agent)
{ "type": "wake" }
```

## Hook Events

The notification hook (`notify-agi.sh`) handles multiple Claude Code lifecycle events:

| Event | When | Purpose |
|-------|------|---------|
| `Stop` | Claude finishes responding | Primary completion signal |
| `TaskCompleted` | Task explicitly marked done | Precise completion (Agent Teams) |
| `SessionEnd` | Session terminates | Fallback signal |

Built-in deduplication (`.hook-lock`, 30s window) prevents double notifications.

HTTP hooks are also supported as an alternative — see [Hook Setup](references/hook-setup.md).

## Result Files

All results are written to `data/claude-code-results/`:

| File | Content |
|------|---------|
| `latest.json` | Full result (output, task name, group, timestamp) |
| `task-meta.json` | Task metadata (prompt, workdir, status, cost params) |
| `task-output.txt` | Raw Claude Code stdout |
| `pending-wake.json` | Heartbeat fallback notification |
| `hook.log` | Hook execution log |

## Debugging

```bash
# Watch hook log
tail -f data/claude-code-results/hook.log

# Check latest result
cat data/claude-code-results/latest.json | jq .

# Check task metadata
cat data/claude-code-results/task-meta.json | jq .

# Test Telegram delivery
openclaw message send --channel telegram --target "-5006066016" --message "test"
```

## Gotchas

1. **Must use PTY wrapper** — Direct `claude -p` hangs in exec environments
2. **Hook fires twice** — Stop + SessionEnd both trigger; `.hook-lock` deduplicates (30s window)
3. **Hook stdin is empty in PTY** — Output is read from `task-output.txt`, not stdin
4. **tee pipe race** — Hook sleeps 1s to wait for pipe flush before reading output
5. **Meta freshness** — Hook validates meta age (<2h) and session ID to avoid stale notifications
6. **Agent Teams cost** — Multi-agent tasks use significantly more tokens; always use `--max-budget-usd`
7. **Rate limits** — Claude Code has daily rate limits (reset at 11:00 UTC); the stop hook still fires with `status=done`, making it look like success

## Prompt Tips

See [Prompt Guide](references/prompt-guide.md) for examples and best practices.

## License

MIT
