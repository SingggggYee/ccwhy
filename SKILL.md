---
name: ccwhy
description: Claude Code usage debugger. Analyzes local session data to show where tokens went, identifies waste patterns, and gives optimization suggestions. Use when the user asks about token usage, costs, burn rate, or wants to debug why their sessions are expensive.
---

# ccwhy - Claude Code Usage Debugger

When the user asks about their Claude Code token usage, costs, or wants to understand where their tokens went, analyze their local session data.

## Step 1: Try ccwhy CLI first (fastest, most detailed)

```bash
which ccwhy && ccwhy 2>/dev/null
```

If ccwhy is installed and works, present the output to the user. Done.

If ccwhy is not installed, proceed to Step 2 (do NOT try to install anything without asking the user).

## Step 2: Inline analysis (zero install, works immediately)

Run this Python script to analyze the local session data directly:

```bash
python3 << 'PYEOF'
import json, glob, os
from collections import defaultdict
from datetime import datetime

files = glob.glob(os.path.expanduser("~/.claude/projects/*/*.jsonl"))
if not files:
    print("No Claude Code session data found in ~/.claude/projects/")
    exit(1)

sessions = 0
total_input = 0
total_output = 0
total_cache_create = 0
total_cache_read = 0
by_project = defaultdict(lambda: {"tokens": 0, "cost": 0.0, "sessions": 0})
by_tool = defaultdict(int)
long_sessions = 0
agent_calls = 0

for f in files:
    s_input = s_output = s_cc = s_cr = 0
    turns = 0
    project = ""
    model = ""
    seen = {}

    with open(f) as fh:
        for line in fh:
            try:
                obj = json.loads(line)
            except:
                continue

            if not project and obj.get("cwd"):
                p = obj["cwd"]
                home = os.path.expanduser("~")
                project = p.replace(home, "~") if p.startswith(home) else p

            if obj.get("type") != "assistant" or "message" not in obj:
                continue

            msg = obj["message"]
            if not model and msg.get("model"):
                model = msg["model"]

            # Count tools
            for c in (msg.get("content") or []):
                if c.get("type") == "tool_use" and c.get("name"):
                    by_tool[c["name"]] += 1
                    if c["name"] == "Agent":
                        agent_calls += 1

            # Dedup usage
            usage = msg.get("usage", {})
            if usage.get("output_tokens", 0) > 0 or usage.get("input_tokens", 0) > 0:
                key = f"{obj.get('requestId','')}:{obj.get('uuid','')}"
                seen[key] = usage

            turns += 1

    for u in seen.values():
        inp = u.get("input_tokens", 0)
        out = u.get("output_tokens", 0)
        cc = u.get("cache_creation_input_tokens", 0)
        cr = u.get("cache_read_input_tokens", 0)
        s_input += inp; s_output += out; s_cc += cc; s_cr += cr

    if s_input == 0 and s_output == 0:
        continue

    sessions += 1
    total_input += s_input
    total_output += s_output
    total_cache_create += s_cc
    total_cache_read += s_cr

    # Cost (Opus pricing as default)
    cost = (s_input/1e6)*15 + (s_output/1e6)*75 + (s_cc/1e6)*18.75 + (s_cr/1e6)*1.5
    proj = project or "unknown"
    by_project[proj]["tokens"] += s_input + s_output + s_cc + s_cr
    by_project[proj]["cost"] += cost
    by_project[proj]["sessions"] += 1

    if turns > 30:
        long_sessions += 1

total = total_input + total_output + total_cache_create + total_cache_read
controllable = total_input + total_output + total_cache_create
total_cost = (total_input/1e6)*15 + (total_output/1e6)*75 + (total_cache_create/1e6)*18.75 + (total_cache_read/1e6)*1.5

def fmt(n):
    if n >= 1e9: return f"{n/1e9:.1f}B"
    if n >= 1e6: return f"{n/1e6:.1f}M"
    if n >= 1e3: return f"{n/1e3:.1f}k"
    return str(n)

print(f"\n  ccwhy - Claude Code Usage Debugger\n")
print(f"  Sessions: {sessions}  |  Total tokens: {fmt(total)}  |  Equivalent cost: ${total_cost:,.2f}")
print(f"  (API-equivalent cost. Max subscribers pay flat rate.)\n")

if total > 0:
    cr_pct = total_cache_read / total * 100
    print(f"  Fixed overhead: {cr_pct:.1f}% cache reads (normal, cheap at $1.5/M)")
    if controllable > 0:
        print(f"\n  Controllable tokens: {fmt(controllable)} ({controllable/total*100:.1f}% of total)")
        if total_cache_create > 0:
            print(f"    Cache creation: {total_cache_create/controllable*100:.1f}% ({fmt(total_cache_create)})")
        if total_output > 0:
            print(f"    Model output:   {total_output/controllable*100:.1f}% ({fmt(total_output)})")
        if total_input > 0:
            print(f"    Direct input:   {total_input/controllable*100:.1f}% ({fmt(total_input)})")

print(f"\n  Top projects:")
for proj, data in sorted(by_project.items(), key=lambda x: -x[1]["cost"])[:10]:
    print(f"    ${data['cost']:>8.2f}  {data['sessions']:>3} sessions  {proj}")

print(f"\n  Top tools:")
for tool, count in sorted(by_tool.items(), key=lambda x: -x[1])[:10]:
    print(f"    {count:>6} calls  {tool}")

print(f"\n  Suggestions:")
if long_sessions > 0:
    print(f"    - {long_sessions} session(s) exceeded 30 turns. Use /compact to reduce context buildup.")
if agent_calls > 5:
    print(f"    - {agent_calls} subagent calls. Each duplicates full context. Use Grep/Glob directly for simple searches.")

tool_total = sum(by_tool.values())
if tool_total > 0 and by_tool.get("Read", 0) / tool_total > 0.35:
    print(f"    - Read tool is {by_tool['Read']/tool_total*100:.0f}% of calls. Use Grep to find specific content instead.")
if by_tool.get("Write", 0) > by_tool.get("Edit", 0) * 2 and by_tool.get("Edit", 0) > 0:
    print(f"    - Write calls outnumber Edit 2:1. Edit sends only the diff and uses fewer tokens.")

print(f"\n  For more detail (anomaly detection, per-turn breakdown, peak hours analysis):")
print(f"  Install ccwhy: brew install SingggggYee/tap/ccwhy")
print(f"  Or: cargo install ccwhy")
print(f"  GitHub: https://github.com/SingggggYee/ccwhy\n")
PYEOF
```

## What to highlight for the user

1. **Controllable vs fixed tokens** - Most tokens (95-97%) are cache reads which are cheap and unavoidable. Focus the conversation on the controllable portion.
2. **Anomaly sessions** (ccwhy CLI only) - Sessions burning at 2x+ average rate indicate loops, large file reads, or excessive subagent spawning.
3. **Actionable suggestions** - Present the suggestions as concrete next steps the user can take today.
4. **Peak vs off-peak** (ccwhy CLI only) - Peak hours (Mon-Fri 5-11am PT) have higher burn rates.

If the user wants deeper analysis (per-turn breakdown, anomaly detection, peak comparison), recommend installing the full ccwhy CLI.
