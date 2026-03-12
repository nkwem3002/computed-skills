# Computed Skills

## What's a skill?

A skill is a markdown file that tells an AI agent what to do. You write instructions, the agent follows them.

```markdown
# SKILL.md

When reviewing code:
1. Check for bugs
2. Check for security issues
3. Check for consistency
```

This works. The agent reads it, does what it says. Every time, the same instructions.

## The problem

The same instructions fire no matter what.

Changed 2 login files? "Check for bugs, security, consistency."
Changed 40 config files? "Check for bugs, security, consistency."

The agent can't prioritize because you didn't prioritize. You wrote one set of instructions for every possible situation.

It gets worse when skills manage data. Say your agent tracks mistakes it made — 87 entries across 3 files. Every session, the agent re-reads all 1,200 lines to check for duplicates before logging a new one. That's the agent doing data parsing. Slowly. Expensively. When Python could do it in milliseconds.

## Computed skills

What if the skill was a program that *generates* the instructions?

```
Skill (static):    Markdown ───────────────────────→ Agent
Skill (computed):  Markdown → Script → Markdown ──→ Agent
```

The agent still receives markdown. That doesn't change. But the markdown comes from a script that looked at the situation first.

The SKILL.md becomes a one-liner that calls the script:

```yaml
---
name: code-review
description: Context-aware code review
---

!`python3 ${CLAUDE_SKILL_DIR}/scripts/generate.py $ARGUMENTS`
```

The `!`command`` syntax runs the script before the agent sees anything. The script prints markdown to stdout. That output becomes the instructions.

## What the script does

The script is where the thinking happens. It can:

**Read context.** What files changed? Are they security-sensitive? Is this a big refactor or a small fix?

**Pick a strategy.** 2 auth files → focus on security. 40 config files → focus on consistency. Test-only changes → focus on correctness.

**Remember past runs.** A JSON file tracks what happened last time. Used the same strategy twice? Add a "fresh eyes" pass. Past reviews missed error handling? Weight that higher.

**Pre-digest data.** Instead of the agent parsing 1,200 lines of structured entries, the script does it and hands the agent a summary: "4 entries need promotion, 12 are stale, here's the next sequence number."

The agent gets instructions tailored to right now, not instructions written for every possible situation.

## Example: code review

A static code review skill gives the same checklist every time. The computed version reads git and adapts.

**2 auth files changed:**
```
Review Strategy: deep (line-by-line)
Signals: security-sensitive files detected

Lens weights:
- Safety       ██████████░░░░░░░░░░ 50%  ← focus here
- Correctness  ██████░░░░░░░░░░░░░░ 30%
- Robustness   ███░░░░░░░░░░░░░░░░░ 15%
- Consistency  █░░░░░░░░░░░░░░░░░░░  5%

⚠ Security Alert: check for hardcoded secrets, .env exposure
```

**22 config files changed:**
```
Review Strategy: architectural (forest over trees)
Signals: config-heavy change, possible refactor

Lens weights:
- Consistency  ███████░░░░░░░░░░░░░ 35%  ← focus here
- Correctness  ██████░░░░░░░░░░░░░░ 30%
- Robustness   ████░░░░░░░░░░░░░░░░ 20%
- Safety       ███░░░░░░░░░░░░░░░░░ 15%
```

Same skill. Different instructions. Because the script looked at the git diff before generating them.

## Example: learning capture

An agent that tracks its own mistakes across sessions. Static version: 214 lines of instructions telling the agent how to parse entries, find duplicates, count recurrences.

Computed version: Python parses everything, the agent gets a summary.

**What the agent used to do (every session):**
- Read 1,200 lines of structured entries
- Scan for matching keys manually
- Count recurrences by reading text
- Find entries due for promotion
- Check for stale entries

**What Python does now (in <100ms):**
- Parses all entries into a hash index
- Finds promotion candidates (recurrence >= 2)
- Flags stale entries (>21 days, never recurred)
- Computes next entry sequence numbers
- Does exact + fuzzy matching on duplicate keys

The agent gets: "Here are the 4 entries due for promotion, here are the 12 stale entries, here's the Pattern-Key index. Now make the judgment calls."

## When to use which

**Static skills** are the right default. They're simple, readable, anyone can edit them.

Use them when: instructions don't change. Style guides. Deploy checklists. Commit formats.

**Computed skills** are for when static breaks down.

Use them when:
- What to do depends on what's happening right now
- The agent parses structured data that code could pre-digest
- You want the skill to remember past runs
- Different contexts need different strategies

## How to build one

1. Write the skill in static markdown first. Get the instructions right.
2. Notice what changes between invocations. What context matters?
3. Write a script that generates the markdown based on that context.
4. Replace the SKILL.md body with the `!`command`` call.
5. Add a state file (JSON) if you want memory across runs.

The script outputs markdown to stdout. That's the whole interface. No framework, no SDK — just print what you want the agent to read.

## What you need

- Python 3.8+ (or any language that prints to stdout)
- A skill system that supports `!`command`` or equivalent
- Works with [Claude Code](https://claude.ai/claude-code) and [OpenClaw](https://openclaw.com)

## License

MIT
