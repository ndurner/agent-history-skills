# Agent History Skills

Codex skills for locating and interpreting local history from AI agent tools.

The project starts with Claude because Claude Code, Claude Desktop, and Claude Cowork use several different local stores for session metadata, transcripts, audit logs, drafts, and Chromium app state. The goal is to make those stores discoverable without exposing private machine-specific details.

## Skills

### `claude-history`

Locate local conversation history for Claude Code, Claude Desktop, and Claude Cowork.

Use it for questions like:

- Where does Claude Code store session transcripts?
- Where does Claude Cowork store `audit.jsonl` logs?
- Search local Claude history for a sentinel phrase.
- Prepare a privacy-safe report of local Claude transcript and metadata locations.

Path:

```text
claude-history/SKILL.md
```

## Scope

These skills are for local history discovery and interpretation. They should default to read-only inspection and should avoid printing secrets, cookies, OAuth tokens, credentials, account IDs, organization IDs, or large transcript excerpts.

Future skills may cover Codex, opencode, Amp, and other agent tools as their local history formats are mapped.

## Layout

```text
agent-history-skills/
  README.md
  claude-history/
    SKILL.md
```

Each skill should be self-contained. Add scripts or references only when they materially improve repeatability; prefer a single `SKILL.md` when the workflow can be expressed cleanly there.
