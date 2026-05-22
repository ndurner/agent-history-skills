# Agent History Skills

Skills for letting AI agents responsibly use their own local session history as memory.

Modern agent tools leave useful evidence behind: JSONL transcripts, audit logs, tool calls, generated artifacts, browser state, cached drafts, and lightweight prompt indexes. That history can answer questions like "what did we already try?", "where did this finding come from?", and "which old run produced that artifact?" But it can also be huge and private.

This repository turns hard-won local recovery workflows into reusable skills. It is a set of read-only playbooks for finding the right local history file, querying it narrowly, extracting defensible evidence, and avoiding accidental disclosure of secrets or private machine details.

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
claude-history.md
```

### `codex-history`

Locate and query local Codex session transcripts.

Use it for questions like:

- Search prior Codex sessions for a phrase or project.
- Recover a finding from a previous Codex thread.
- Extract tool calls or final answers from a Codex JSONL transcript.
- Prepare a privacy-safe evidence note from earlier Codex work.

Path:

```text
codex-history.md
```

## Design Principles

- Prefer sentinel searches and metadata over opening whole transcripts.
- Treat local agent history as private research material by default.
- Capture source session, question, finding, and evidence when using past work.
- Distinguish durable transcripts from caches, drafts, logs, and browser state.
- Verify claims against artifacts when the task is visual or behavioral; transcript metadata is not enough.

## Scope

These skills are for local history discovery and interpretation. They should default to read-only inspection and should avoid printing secrets, cookies, OAuth tokens, credentials, account IDs, organization IDs, raw HMACs, or large transcript excerpts.

Future skills may cover opencode, Amp, and other agent tools as their local history formats are mapped.

## Layout

```text
agent-history-skills/
  README.md
  claude-history.md
  codex-history.md
```

Each skill should be self-contained. Add scripts or references only when they materially improve repeatability; prefer a single Markdown file when the workflow can be expressed cleanly there.
