---
name: seance
description: Commune with past Claude Code sessions. Use for recalling past work, exploring session history, or resurrecting abandoned work. Invoke with `/seance` for quick list, `/seance <question>` for exploration, or `/seance resurrect <target>` for actionable replan.
allowed-tools: [Bash, Read, Write, Task]
---

# Seance

Commune with your past sessions.

## Routing

| Input | Action |
|-------|--------|
| `/seance` | Quick list of recent sessions (instant, no LLM) |
| `/seance resurrect <target>` | Oracle with **action intent** → generates replan |
| `/seance <anything else>` | Oracle with **info intent** → explores and answers |

All paths except quick list use the same Oracle engine. The word "resurrect" signals action intent.

## Path Resolution

```bash
ENCODED_PATH=$(echo "$PWD" | sed 's|/|-|g')
SESSION_DIR="$HOME/.claude/projects/$ENCODED_PATH"
INDEX_FILE="$SESSION_DIR/sessions-index.json"
```

---

## Quick List

When user runs `/seance` with no arguments:

```bash
jq -r '.entries | sort_by(.modified) | reverse | .[:10][] |
  "\(.modified | .[0:10])  \(.sessionId | .[0:8])  \(.messageCount // 0 | tostring | if length < 3 then " " * (3 - length) + . else . end) msgs  \(.summary // .firstPrompt // "No summary" | .[0:55])"' \
  "$INDEX_FILE" 2>/dev/null || echo "No sessions found for this project"
```

Present as:
```
DATE        ID        MSGS  SUMMARY
2026-01-30  e0f76681   46   Fix Missing AssuranceConfig on Organization Creation
2026-01-29  656f96e1   11   Mercury child org tests hidden permissions
...

Try: "/seance what was I working on?" or "/seance resurrect <id>"
```

---

## Oracle

Oracle handles both exploration (`/seance <question>`) and resurrection (`/seance resurrect <target>`). Same engine, different intent.

### Intent Detection

| Input | Intent | Oracle Output |
|-------|--------|---------------|
| `/seance what was session X about?` | Info | Summary, explanation |
| `/seance why does auth keep breaking?` | Info | Analysis, patterns |
| `/seance resurrect abc123` | Action | Replan to continue work |
| `/seance resurrect #42` | Action | PR status + next steps |
| `/seance resurrect feature-branch` | Action | Branch analysis + completion plan |

**Resurrect targets:**
- Session ID (or prefix): `resurrect abc123`
- PR number: `resurrect #42` or `resurrect PR:42`
- Branch name: `resurrect feature-branch`

### Time Ambiguity Check

For info-intent questions, check if time scope is ambiguous:

| Question | Action |
|----------|--------|
| "What was I working on?" | Ask: "Recent, last month, or all time?" |
| "What was I working on last week?" | Proceed (explicit) |
| "Why does auth keep breaking?" | Search all (no time constraint) |

### Dispatch Haiku Subagent

```
Task:
  subagent_type: "general-purpose"
  model: "haiku"
  prompt: <see template below>
```

### Subagent Prompt Template

```
You are a session log analyst. Your job is to explore Claude Code session history and return structured findings.

## Environment

Session index: $INDEX_FILE
Session logs: $SESSION_DIR/*.jsonl
Current directory: $PWD

## User's Request

"<USER_INPUT>"

## Intent

<INFO | ACTION>

If ACTION (resurrect): Your goal is to help the user continue this work. Gather context, check current state, and create an actionable plan.

If INFO: Your goal is to answer the user's question with evidence.

## Available Queries

**Count all sessions:**
```bash
jq '.entries | length' "$INDEX_FILE"
```

**List ALL sessions:**
```bash
jq -r '.entries | sort_by(.modified) | reverse' "$INDEX_FILE"
```

**Filter by date range:**
```bash
jq --arg since "2026-01-20" '.entries | map(select(.modified >= $since)) | sort_by(.modified) | reverse' "$INDEX_FILE"
```

**Search by keyword:**
```bash
jq --arg q "<term>" '.entries | map(select((.summary // "" | ascii_downcase | contains($q | ascii_downcase)) or (.firstPrompt // "" | ascii_downcase | contains($q | ascii_downcase)))) | sort_by(.modified) | reverse' "$INDEX_FILE"
```

**Get session metadata:**
```bash
jq --arg id "<session_id>" '.entries[] | select(.sessionId | startswith($id))' "$INDEX_FILE"
```

**Read session turns:**
```bash
jq -c 'select(.type == "user" or .type == "assistant")' "$SESSION_DIR/<session_id>.jsonl" | head -30
```

**Get files modified in session:**
```bash
jq -r 'select(.type == "assistant") | .message.content[]? | select(.type == "tool_use") | select(.name == "Edit" or .name == "Write") | .input.file_path // .input.path // empty' "$SESSION_DIR/<session_id>.jsonl" | sort -u
```

**Get errors from session:**
```bash
jq -r 'select(.type == "user") | .message.content[]? | select(.type == "tool_result" and .is_error == true) | .content | .[0:150]' "$SESSION_DIR/<session_id>.jsonl"
```

**For PR resurrection - get PR info:**
```bash
gh pr view <number> --json title,state,body,reviews,comments,statusCheckRollup
```

**For branch resurrection - get branch status:**
```bash
git log main..<branch> --oneline
git diff main..<branch> --stat
```

## Output Schema

Return ONLY a JSON object wrapped in <result> tags.

### For INFO intent:

<result>
{
  "intent": "info",
  "answer": "Direct answer (2-3 sentences)",
  "scope": {
    "total_sessions": 42,
    "sessions_searched": 42,
    "date_range": "2026-01-01 to 2026-01-30"
  },
  "sessions": [
    {
      "id": "session-id",
      "date": "YYYY-MM-DD",
      "summary": "What this session was about",
      "relevance": "Why it matters to the question"
    }
  ],
  "evidence": [
    {
      "session_id": "abc123",
      "turn": 7,
      "quote": "Relevant excerpt (max 100 chars)"
    }
  ],
  "follow_up": "Suggested next question"
}
</result>

### For ACTION intent (resurrect):

<result>
{
  "intent": "action",
  "target_type": "session | pr | branch",
  "target_id": "abc123 | #42 | feature-branch",
  "original_goal": "What was being worked on",
  "what_was_done": [
    "Completed item 1",
    "Completed item 2"
  ],
  "current_state": {
    "status": "Description of current state",
    "blockers": ["Any blockers or issues"],
    "files_touched": ["file1.ts", "file2.ts"]
  },
  "plan": [
    "Next step 1",
    "Next step 2",
    "Next step 3"
  ],
  "evidence": [
    {
      "session_id": "abc123",
      "turn": 7,
      "quote": "Relevant excerpt"
    }
  ]
}
</result>

## Constraints

- **Search ALL sessions by default** unless user specifies time range
- **Always report scope** for info queries
- Read at most 30 turns per session
- Return at most 5 sessions, prioritize relevance
- Always include evidence citations
- No markdown or prose outside JSON
```

### Processing Response

**For INFO intent:**

```
## What I Found

<answer>

_Searched <total> sessions from <date_range>_

### Relevant Sessions

| Date | ID | Summary |
|------|-----|---------|
...

### Evidence

<citations>

### You might also ask...

<follow_up>
```

**For ACTION intent (resurrect):**

```
## Resurrecting: <target>

### Original Goal
<original_goal>

### What Was Done
<what_was_done as bullets>

### Current State
<status>
- Files: <files_touched>
- Blockers: <blockers>

### Plan to Continue
<plan as numbered steps>

Ready to start? Just say "go" or ask me to adjust the plan.
```

---

## Error Handling

**No sessions:**
```
No sessions found for this project.
Try running from a directory where you've used Claude Code.
```

**Target not found:**
```
Couldn't find <target>.
Run `/seance` to see available sessions, or check the PR/branch exists.
```

**Invalid JSON from subagent:**
```
I explored your sessions but had trouble structuring the results:

<raw response>
```

---

## Secrets Scrubbing

When displaying session content:

```bash
sed -E \
  -e 's/([A-Za-z_]*(KEY|TOKEN|SECRET|PASSWORD|API_KEY)[^=]*[=:][[:space:]]*)[A-Za-z0-9_\-]{16,}/\1[REDACTED]/gi' \
  -e 's/Bearer [A-Za-z0-9_\-\.]{20,}/Bearer [REDACTED]/g' \
  -e 's/sk-[A-Za-z0-9]{32,}/sk-[REDACTED]/g' \
  -e 's/ghp_[A-Za-z0-9]{36}/ghp_[REDACTED]/g'
```

---

## Quick Reference

```
/seance                      → List recent sessions
/seance <question>           → Ask anything about your history
/seance resurrect <session>  → Replan to continue session work
/seance resurrect #42        → Replan to finish PR
/seance resurrect <branch>   → Replan to complete branch
```
