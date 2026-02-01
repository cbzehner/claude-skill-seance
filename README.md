# Seance

Commune with your past Claude Code sessions.

## What It Does

Seance explores your session history through conversation. Ask questions about past work, find patterns, or resurrect abandoned tasks with an actionable plan.

## Usage

```
/seance                      → Quick list of recent sessions
/seance <question>           → Oracle explores and answers
/seance resurrect <target>   → Replan to continue work
```

## Examples

```
/seance
→ Shows your 10 most recent sessions

/seance what was I working on last week?
→ Searches sessions, synthesizes a summary

/seance why does the auth module keep breaking?
→ Finds error patterns across sessions

/seance resurrect abc123
→ Analyzes session, creates plan to continue

/seance resurrect #42
→ Checks PR status, plans next steps

/seance resurrect feature-branch
→ Reviews branch progress, plans completion
```

## How It Works

**Quick List**: Direct jq query, instant.

**Oracle**: Spawns a Haiku subagent that explores session logs, returns structured findings. Main thread synthesizes into a conversational answer with citations.

**Resurrect**: Same Oracle engine, but with action intent. Gathers context from session/PR/branch, checks current state, generates an actionable plan to continue.

## Installation

```bash
git clone https://github.com/cbzehner/claude-skill-seance ~/.claude/skills/seance
```

Or symlink:

```bash
ln -s /path/to/claude-skill-seance ~/.claude/skills/seance
```

## Requirements

- `jq` for JSON parsing (`brew install jq`)
- `gh` for PR resurrection (`brew install gh`)
- Claude Code session logs (created when you use Claude Code)

## License

MIT
