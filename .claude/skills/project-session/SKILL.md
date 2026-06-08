---
name: project-session
description: Use when starting or tracking a coding agent session in a project; creates or updates project-local .codex/session and .codex/session.log (or .claude/session and .claude/session.log) files.
---

# Project Session Tracking Skill

## Purpose

Every coding-agent session must keep a project-local session record.

Session records are used to understand what each agent worked on in the current project. These files are local runtime state and must not be committed to Git.

This rule applies to:

- Codex / OpenAI coding agents
- Claude Code
- Other compatible coding agents that follow this instruction file

## Project-local rule

All session files must be created relative to the current project root.

Do not write these files to the user's home directory, a global config directory, or another shared location.

Each project must have its own independent session records.

## Session files

| Agent | Current session file | Session log file |
| --- | --- | --- |
| Codex | `.codex/session` | `.codex/session.log` |
| Claude Code | `.claude/session` | `.claude/session.log` |

## Required behavior on session start

When a new agent session starts, the agent must:

1. Detect the project root.
2. Identify which agent is running.
3. Create the required agent directory if it does not exist.
4. Create or update the current session file.
5. Generate a session ID if one does not already exist for the current session.
6. Determine a short session topic from the user's initial request.
7. Add or update one line in the agent's session log using this format:

```text
<session-id> - <session-topic>
```
