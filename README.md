# Computed Skills

**Skills that adapt to what's actually happening.**

A static skill gives the same instructions every time. A computed skill runs a script first — analyzes the situation, pre-digests data, picks a strategy — then hands the agent tailored markdown. The agent never knows a script was involved.

```
Static:    SKILL.md ──────────────────────────→ Agent reads markdown
Computed:  SKILL.md → runs script → markdown ─→ Agent reads markdown
```

Works with [Claude Code](https://claude.ai/claude-code) and [OpenClaw](https://openclaw.ai). Any language that prints to stdout.

## The problem

A static skill for daily planning:

```markdown
# Daily Focus

Review your calendar and todo list. Identify the top 3 priorities.
Consider deadlines, energy levels, and blocked tasks.
```

The agent gets this whether you have 0 todos or 47. Whether it's Monday morning or Friday at 5pm. Whether you missed yesterday's deadlines or cleared everything.

A computed version:

```python
# generate.py
import json
from datetime import datetime
from pathlib import Path

def generate():
    todos = json.loads(Path("todos.json").read_text())
    overdue = [t for t in todos if t["due"] < datetime.now().isoformat() and not t["done"]]
    today = [t for t in todos if t["due"][:10] == datetime.now().strftime("%Y-%m-%d")]

    if overdue:
        print(f"# ⚠️ {len(overdue)} Overdue Tasks\n")
        print("**Handle these first:**")
        for t in overdue:
            print(f"- {t['title']} (due {t['due'][:10]})")
        print()

    if today:
        print(f"## Today's {len(today)} Tasks\n")
        for t in today:
            print(f"- {'~~' + t['title'] + '~~' if t['done'] else t['title']}")
    elif not overdue:
        print("# Clear day — no tasks due. Good time for deep work.")

generate()
```

Now the agent sees "⚠️ 3 Overdue Tasks" on a bad day and "Clear day" on a good one. Same skill, different instructions.

## Quick start

**1. Create the skill:**

```
my-skill/
├── SKILL.md
└── scripts/
    └── generate.py
```

**2. SKILL.md** (the thin shell):

```yaml
---
name: my-skill
description: What it does and when to trigger it
---

!`python3 ${CLAUDE_SKILL_DIR}/scripts/generate.py $ARGUMENTS`
```

**3. generate.py** (the brain):

```python
#!/usr/bin/env python3
import os, sys

def main():
    args_str = os.environ.get("ARGUMENTS", "").strip()
    args = args_str.split() if args_str else sys.argv[1:]
    mode = args[0] if args else ""

    if mode == "status":
        print("# Status Dashboard\n")
        # ... verbose output for manual invocation
    else:
        print("# Default Mode\n")
        # ... context-aware instructions

if __name__ == "__main__":
    main()
```

That's it. The `!`command`` syntax is a preprocessing directive — it runs before the agent sees anything and replaces itself with stdout.

## Real example: self-improve

This is from a production agent that tracks its own mistakes across 87+ entries. The static version was 214 lines of instructions telling the agent how to parse entries and count duplicates. The computed version:

```python
# What Python does in <100ms:
entries = parse_all_entries()           # Parse 1,200 lines of markdown
dupes = find_duplicates(new_key)        # Exact + fuzzy matching
promotable = [e for e in entries if e["recurrence"] >= 2]
stale = [e for e in entries if days_since(e["date"]) > 30]
```

```
# What the agent sees:

## Self-Improve Status

**LEARNINGS** — 68 total, 61 open
**ERRORS** — 23 total, 21 open

## Due for Promotion (Recurrence >= 2)
- [workspace-path-confusion] Count: 2 — verify paths before operations

## Stale (>30 days, no recurrence)
- [whatsapp-gateway-patterns] — resolved, service removed
```

The agent focuses on judgment — *should* this be promoted? *is* this still relevant? — not data parsing.

**Full source:** [`examples/self-improve/scripts/generate.py`](examples/self-improve/scripts/generate.py)

## Three patterns

### 1. Computed — script generates context-aware instructions

The script reads the environment and outputs different instructions.

```
2 auth files changed  →  "Security Review: check for secrets, validate auth"
40 config files       →  "Config Audit: check YAML syntax, look for breaking changes"
```

Best for: code review, data analysis, anything where context determines strategy.

**Example:** [`examples/smart-review`](examples/smart-review)

### 2. Computed-static hybrid — conditional plain English

The script wraps behavioral instructions in conditionals. The agent only sees the relevant branch.

```python
hours_since = check_staleness()

if hours_since > 168:  # 7+ days
    print("# ATTENTION: You haven't tracked decisions in 7 days.")
    print("Initialize now: ...")
    print("Then add nodes as conversation evolves.")
elif hours_since > 24:
    print("Reminder: decision tracking is due.")
else:
    print("Decision tracking is current.")
```

This solves a real problem: agents ignore "always-on" mandates buried in long static documents. The hybrid makes the LLM only see what's urgent right now.

Best for: always-on behaviors, escalating reminders, conditional workflows.

### 3. Autonomous dispatch (OpenClaw)

On [OpenClaw](https://openclaw.ai), computed skills can be dispatched by a system cron without any LLM deciding to run them:

```
System cron (every 30m)
  → Python calls generate.py to pick tasks
  → POSTs to /hooks/agent with task prompt
  → Agent runs on cheap model, writes results
```

The scheduling is deterministic (Python). The execution uses an LLM. The LLM never decides *whether* to work — only *how*.

This doesn't apply to Claude Code (no webhook API). The computed skill itself works the same on both platforms — only the dispatch layer is OpenClaw-specific.

## Multi-mode convention

Most production computed skills support multiple modes via arguments:

| Mode | Purpose | Output |
|------|---------|--------|
| *(no args)* | Default / always-on | Context-aware instructions |
| `status` | Manual dashboard (`/command`) | Verbose report |
| `heartbeat` | Periodic health check | Silent unless problems found |

The `heartbeat` convention matters: **output nothing when everything is OK.** Only speak up when there's something to act on. Silence means healthy.

```yaml
# SKILL.md — user can invoke with /my-skill or /my-skill status
!`python3 ${CLAUDE_SKILL_DIR}/scripts/generate.py $ARGUMENTS`
```

## When to use what

| Situation | Pattern |
|-----------|---------|
| Instructions never change | Static (plain markdown) |
| Instructions depend on context (git state, file contents, time) | Computed |
| Behavioral instructions that get ignored when too long | Hybrid |
| Agent needs to parse structured data (logs, entries, state files) | Computed |
| Skill should remember across runs | Computed + state file |
| Skill should run autonomously on a schedule | Computed + dispatch (OpenClaw) |

**Start static, switch to computed when you notice the agent doing work that code could do faster.**

## How to build one

1. **Write the skill as static markdown first.** Get the instructions right.
2. **Notice what changes between invocations.** What context matters?
3. **Write a script that generates the markdown.** Read context, make decisions, print to stdout.
4. **Replace the SKILL.md body** with the `!`command`` invocation.
5. **Add a state file** (JSON) if you want memory across runs.

### Passing arguments

`$ARGUMENTS` in SKILL.md is text substitution — replaced with invocation arguments *before* the shell runs.

**Important:** On OpenClaw 2026.3.12+, the scanner flags `$ARGUMENTS` in Python files (even in comments). Keep `$ARGUMENTS` in SKILL.md only. In Python, use:

```python
args_str = os.environ.get("ARGUMENTS", "").strip()
args = args_str.split() if args_str else sys.argv[1:]
```

### Error handling

If the script crashes, the agent gets nothing (or a traceback). For production skills:

```python
try:
    generate()
except Exception as e:
    print("# Fallback Instructions\n\nCheck for bugs and consistency.")
    print(f"\n<!-- generator error: {e} -->")
```

## Examples

Three working examples from a production agent running 15 skills (11 computed) on a 24/7 autonomous system:

| Example | Pattern | What it does |
|---------|---------|-------------|
| [`smart-review`](examples/smart-review) | Computed | Reads git diff, picks review strategy, tracks past runs |
| [`self-improve`](examples/self-improve) | Computed + multi-mode | Parses 87+ entries, detects recurrence, flags promotions |
| [`check-pattern`](examples/check-pattern) | Computed (sub-skill) | Duplicate detection — reuses self-improve's generator |

## Prior art

The [`!`command`` syntax](https://code.claude.com/docs/en/skills#inject-dynamic-context) is documented by Anthropic under "inject dynamic context" but rarely used for full prompt generation.

As of March 2026:

- **[Anthropic's skills repo](https://github.com/anthropics/skills/)** (17 skills) — all static, zero computed
- **[SkillsMP](https://skillsmp.com)** (400K+ indexed) — no category for computed skills
- **[vibereq](https://github.com/dipasqualew/vibereq)** — the only other project we've found using the pattern in production

Related approaches: [DSPy](https://dspy.ai/) (programmatic prompt compilation via SDK), [context engineering](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) (the broader concept).

## Requirements

- Python 3.8+ (or any language that prints to stdout)
- A skill system that supports `!`command`` preprocessing
- Works with [Claude Code](https://claude.ai/claude-code) and [OpenClaw](https://openclaw.ai)

## License

MIT
