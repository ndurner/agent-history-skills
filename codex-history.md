---
name: codex-session-reader
description: Read past local Codex sessions from the user's `.codex` archive, find relevant prior work by keyword, and extract durable evidence without loading huge transcripts into context. Use when the user asks to look up a prior Codex thread, search Codex history, recover earlier findings, or cite what Codex discovered in a past conversation.
---

# Codex Session Reader

Use this when a task depends on earlier Codex work and the user mentions a keyword, past thread, experiment, branch, phrase, or artifact. Work read-only unless the user explicitly asks to export, migrate, delete, or modify history.

## Storage Layout

Codex desktop stores local session transcripts as JSONL:

```bash
~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl
```

Older files may exist directly under `~/.codex/sessions/`:

```bash
~/.codex/sessions/rollout-*.jsonl
```

Prompt history is useful for orientation but is not the full transcript:

```bash
~/.codex/history.jsonl
```

Browser side-panel state is separate:

```bash
~/.codex/browser/sessions/*.toml
```

Those TOML files identify browser-side sessions but do not replace the JSONL Codex transcript.

## Workflow

Start with targeted search. Do not open a large JSONL transcript directly unless it is already known to be small.

```bash
rg -n "Godspeed|XNNPACK|PhoneClaw|grid overlay" ~/.codex/sessions -g '*.jsonl'
```

If the keyword is broad, return paths first:

```bash
rg -l "XNNPACK" ~/.codex/sessions -g '*.jsonl'
```

Then inspect file size and modification time:

```bash
ls -lh /path/to/rollout-*.jsonl
wc -l /path/to/rollout-*.jsonl
```

Some transcripts exceed 100 MB. Treat them as data sources to query, not files to paste into context.

## Find Candidate Sessions

Recent sessions:

```bash
find ~/.codex/sessions -type f -name '*.jsonl' -print0 | xargs -0 ls -lt | head -40
```

Sessions from a date:

```bash
find ~/.codex/sessions/2026/05/14 -type f -name '*.jsonl' -print
```

Prompt history:

```bash
rg -n "Godspeed|XNNPACK|PhoneClaw" ~/.codex/history.jsonl
```

Session metadata:

```bash
jq -r 'select(.type=="session_meta") | [.payload.timestamp, .payload.cwd, .payload.id, .payload.model_provider] | @tsv' /path/to/session.jsonl
```

## Read A Session Safely

For a quick human-readable transcript, use `event_msg` entries first. They usually contain user-facing user and assistant messages without every tool payload.

```bash
jq -r '
  select(.type=="event_msg" and (.payload.type=="user_message" or .payload.type=="agent_message"))
  | "[" + .timestamp + "] " + .payload.type + ":\n" + (.payload.message // "") + "\n"
' /path/to/session.jsonl | sed -n '1,240p'
```

To extract model messages from `response_item` records:

```bash
jq -r '
  select(.type=="response_item" and .payload.type=="message")
  | (.payload.role // "unknown") as $role
  | .payload.content[]?
  | select(.type=="input_text" or .type=="output_text")
  | $role + ":\n" + .text + "\n"
' /path/to/session.jsonl | sed -n '1,240p'
```

To locate tool calls without reading all outputs:

```bash
jq -r '
  select(.type=="response_item" and .payload.type=="function_call")
  | "[" + .timestamp + "] CALL " + .payload.name + " " + (.payload.arguments | tostring)
' /path/to/session.jsonl
```

To inspect output around a known call id or phrase:

```bash
rg -n "call_abc123|important phrase" /path/to/session.jsonl -C 3
```

## Search Patterns

Use narrow keyword sets and expand only if needed.

Runtime findings:

```bash
rg -n "XNNPACK|LiteRT|crash|weight-cache|CPU-only|send message" ~/.codex/sessions -g '*.jsonl'
```

Visual reasoning work:

```bash
rg -n "Godspeed|sketch|coordinate scaffold|grid|A1-F6|move_text_overlay|overlay" ~/.codex/sessions -g '*.jsonl'
```

Prior work references:

```bash
rg -n "PhoneClaw|OpenPhone|Seeth|MCoT|coordinate scaffolding|C2PA" ~/.codex/sessions -g '*.jsonl'
```

Package or repo work:

```bash
rg -n "aileen-job.yaml|Relay Desk|Desk Mode|Field Mode|handoff package" ~/.codex/sessions -g '*.jsonl'
```

## Summarize Evidence

When using old sessions for a writeup, capture four things:

- `Source session`: absolute JSONL path and date.
- `Question`: what the user or agent was investigating.
- `Finding`: the shortest defensible conclusion.
- `Evidence`: command output, file path, error message, screenshot path, or generated artifact path.

Prefer a small Markdown evidence note in the current repo over relying on memory:

```md
## Finding

- Source session: `/Users/name/.codex/sessions/2026/05/14/rollout-...jsonl`
- Keyword: `XNNPACK`
- Finding: E4B simulator failure occurred during LiteRT session creation, before overlay prompt quality could be evaluated.
- Evidence: crash path mentioned `MMapWeightCacheProvider::OffsetToAddr` under `LlmLiteRtCompiledModelExecutorStatic::Create`.
```

## Privacy And Safety

Past sessions can contain API keys, local paths, private notes, screenshots, copied social posts, and tool outputs. Treat transcripts as local private research material unless the user explicitly asks to publish or share them.

Before copying content into a public writeup:

- Remove secrets, phone numbers, names that are not intentionally public, and exact private paths unless the path itself is relevant.
- Separate quoted source material from your own conclusions.
- Prefer paraphrase for public submission text.
- Preserve exact command output only when it is technical evidence and safe to disclose.

## Common Pitfalls

- Do not assume `history.jsonl` is the full transcript. It is only an index-like prompt history.
- Do not paste a whole large session into the answer. Query it with `rg` and `jq`.
- Do not treat tool-call output as user-facing truth without checking surrounding messages.
- Do not confuse browser session TOML files with Codex transcript JSONL files.
- Do not edit files discovered in past sessions unless the current user request requires it.

## Useful One-Liners

List response item types in a session:

```bash
jq -r 'select(.type=="response_item") | .payload.type' /path/to/session.jsonl | sort | uniq -c
```

List the working directory and model for each candidate:

```bash
for f in ~/.codex/sessions/2026/05/*/*.jsonl; do
  jq -r 'select(.type=="session_meta") | [.payload.timestamp, .payload.cwd, input_filename] | @tsv' "$f"
done
```

Extract user prompts:

```bash
jq -r 'select(.type=="event_msg" and .payload.type=="user_message") | "[" + .timestamp + "] " + .payload.message + "\n"' /path/to/session.jsonl
```

Extract final answers:

```bash
jq -r 'select(.type=="event_msg" and .payload.type=="agent_message" and .payload.phase=="final_answer") | "[" + .timestamp + "] " + .payload.message + "\n"' /path/to/session.jsonl
```
