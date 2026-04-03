---
name: ccwhy
description: Analyze Claude Code token usage. Shows where tokens went, which projects cost most, and how to reduce waste. Use when user asks about token usage, costs, or burn rate.
---

# ccwhy

Debug your Claude Code token usage.

## What it does

Parses local JSONL session files in `~/.claude/projects/` to show where tokens went. Breaks down usage by project, tool, model, and identifies anomaly sessions.

## Data access

- Reads ONLY `~/.claude/projects/*/*.jsonl` (local Claude Code session logs)
- NO network access, NO API keys, NO credentials needed
- All analysis runs offline on your machine
- Source code: https://github.com/SingggggYee/ccwhy

## Usage

```bash
ccwhy
```

If not installed:

```bash
brew install SingggggYee/tap/ccwhy
```

Or: `cargo install ccwhy`

More commands: `ccwhy session <id>`, `ccwhy sessions`, `ccwhy --json`, `ccwhy report --days 7`
