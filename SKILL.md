---
name: claude-usage-analyzer
description: Analyze Claude Code token usage. Shows where tokens went, which projects cost most, and how to reduce waste. Use when user asks about token usage, costs, or burn rate.
config_paths:
  - ~/.claude/projects/*/*.jsonl
requires:
  - ccwhy
---

# Claude Usage Analyzer

Analyze your Claude Code token usage by parsing local session logs.

## Data access

- Reads `~/.claude/projects/*/*.jsonl` (local Claude Code session logs)
- Runs offline, no network access, no API keys, no credentials
- Open source: https://github.com/SingggggYee/ccwhy

## Usage

Requires the `ccwhy` CLI to be pre-installed. See https://github.com/SingggggYee/ccwhy for installation instructions.

```bash
ccwhy
```

More commands:

- `ccwhy report --days 7` (last 7 days)
- `ccwhy sessions` (top sessions by cost)
- `ccwhy session <id>` (per-turn breakdown)
- `ccwhy --json` (machine-readable output)
