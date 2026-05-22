---
name: claude-history
description: Locate and interpret local conversation history for Claude Desktop, Claude Code, and Claude Cowork. Use when the user asks where Claude stores chat transcripts, Claude Code JSONL sessions, Cowork audit logs, local drafts, IndexedDB/LevelDB state, or Claude conversation history; when searching Claude history for a sentinel phrase; or when preparing a privacy-safe report of where Claude conversations live locally.
---

# Claude History

Use this when a task depends on earlier Claude work, a user asks where Claude keeps local history, or the user supplies a sentinel phrase to find in Claude Code, Claude Desktop, or Claude Cowork data.

## Operating Rules

Work read-only unless the user explicitly asks for export, cleanup, migration, or deletion.

Treat transcripts, audit logs, app state, cookies, credentials, OAuth tokens, account IDs, organization IDs, local paths, and uploaded files as sensitive. Do not expose secrets or unnecessary machine-specific details in the final answer.

Report paths with environment-variable forms such as `%USERPROFILE%`, `%APPDATA%`, `%LOCALAPPDATA%`, `$HOME`, or `~` where possible.

Prefer a sentinel supplied by the user. Search exact text first. Use case-insensitive partials only if exact search fails.

Quote conversation content only when needed to prove that a location is correct, and quote the minimum possible excerpt.

## Core Model

Claude history may exist in more than one local store:

- Claude Code CLI: JSONL transcripts under `~/.claude`.
- Claude Desktop: app-profile state, logs, and Chromium storage under the desktop app profile.
- Claude Cowork: local-agent session metadata and `audit.jsonl` logs under `local-agent-mode-sessions`, plus nested `.claude` runtime state inside each local session.

`history.jsonl` is usually a lightweight prompt index. The full Claude Code transcript is usually under `projects/.../*.jsonl`. Cowork `local_*.json` files hold metadata; sibling `local_*` directories hold `audit.jsonl`, output files, and nested runtime state.

Chromium storage can contain useful prompt fragments or drafts even when a regular text search misses them. Search `IndexedDB`, `IndexedDB/*.blob`, `Local Storage/leveldb`, and `Session Storage` with binary-safe search only after durable transcript and metadata stores.

## Storage Layouts

Claude Code:

```text
~/.claude/history.jsonl
~/.claude/projects/<encoded-working-directory>/<session-id>.jsonl
~/.claude/projects/<encoded-working-directory>/<session-id>/subagents/*.jsonl
~/.claude/sessions/*.json
~/.claude/tasks/<task-id>/
~/.claude/todos/*.json
```

Claude Desktop and Cowork on macOS:

```text
~/Library/Application Support/Claude/
~/Library/Application Support/Claude-3p/
~/Library/Application Support/Claude*/local-agent-mode-sessions/<account>/<workspace>/local_*.json
~/Library/Application Support/Claude*/local-agent-mode-sessions/<account>/<workspace>/local_*/audit.jsonl
~/Library/Application Support/Claude*/local-agent-mode-sessions/<account>/<workspace>/local_*/.claude/
~/Library/Application Support/Claude*/IndexedDB/
~/Library/Application Support/Claude*/Local Storage/leveldb/
~/Library/Application Support/Claude*/Session Storage/
~/Library/Caches/claude-cli-nodejs/
```

Claude Desktop and Cowork on Windows:

```text
%USERPROFILE%\.claude\
%APPDATA%\Claude\
%LOCALAPPDATA%\Claude\
%LOCALAPPDATA%\Packages\<ClaudePackage>\LocalCache\Roaming\Claude\
%APPDATA%\Claude\local-agent-mode-sessions\<account>\<workspace>\local_*.json
%APPDATA%\Claude\local-agent-mode-sessions\<account>\<workspace>\local_*\audit.jsonl
%APPDATA%\Claude\claude-code-sessions\<account>\<workspace>\local_*.json
%APPDATA%\Claude\IndexedDB\
%APPDATA%\Claude\IndexedDB\*.blob
%APPDATA%\Claude\Local Storage\leveldb\
%APPDATA%\Claude\Session Storage\
```

Confirm locally; product builds and account types may shift profile names.

## Workflow

1. Identify installed Claude surfaces. Look for `claude`, `~/.claude`, and Claude app profiles under the platform-specific roots above.
2. Search durable transcript stores first: `history.jsonl`, `projects/**/*.jsonl`, `projects/**/subagents/*.jsonl`, `local-agent-mode-sessions/**/audit.jsonl`, `local-agent-mode-sessions/**/local_*.json`, and `claude-code-sessions/**/local_*.json`.
3. Search app state second: logs, `IndexedDB`, `Local Storage/leveldb`, `Session Storage`, and caches.
4. Map metadata to transcripts. Pair a `local_*.json` metadata file with a sibling directory of the same `local_*` id; inspect `audit.jsonl` there for role/content events.
5. Summarize evidence with location class, date, sentinel match, and residual uncertainty. Separate durable transcripts, session metadata, UI drafts/cache, and logs.

## macOS Search

Start narrow:

```bash
find ~/.claude -maxdepth 5 -type f 2>/dev/null | head
find ~/Library/Application\ Support -maxdepth 4 \( -iname '*claude*' -o -iname '*cowork*' \) 2>/dev/null
```

Search Claude Code history. Exact search first:

```bash
rg -n --hidden --no-messages "<sentinel>" ~/.claude -g '*.jsonl' -g '*.json'
```

Search Desktop and Cowork profiles:

```bash
rg -n --hidden --no-messages "<sentinel>" \
  "$HOME/Library/Application Support/Claude" \
  "$HOME/Library/Application Support/Claude-3p" \
  -g '*.jsonl' -g '*.json'
```

If exact search misses, retry selected roots case-insensitively with `-i`.

Search binary Chromium stores only when transcript and metadata search misses:

```bash
rg -a --files-with-matches --hidden --no-messages "<sentinel>" \
  "$HOME/Library/Application Support/Claude/IndexedDB" \
  "$HOME/Library/Application Support/Claude/Local Storage/leveldb" \
  "$HOME/Library/Application Support/Claude/Session Storage" 2>/dev/null
```

## Windows Search

Discover roots:

```powershell
Get-ChildItem -Force $env:USERPROFILE, $env:APPDATA, $env:LOCALAPPDATA |
  Where-Object { $_.Name -match 'claude|anthropic' }

Get-ChildItem -Force "$env:LOCALAPPDATA\Packages" -Directory |
  Where-Object { $_.Name -match 'claude|anthropic' }
```

Search text transcripts:

```powershell
rg -n --hidden --no-messages "<sentinel>" "$env:USERPROFILE\.claude" -g "*.jsonl" -g "*.json"
```

If exact search misses, retry selected roots case-insensitively with `-i`.

Search Desktop and Cowork profiles:

```powershell
rg -n --hidden --no-messages "<sentinel>" "<claude-app-profile-root>" -g "*.jsonl" -g "*.json"
```

Search Chromium stores:

```powershell
rg -a --files-with-matches --hidden --no-messages "<sentinel>" `
  "<claude-app-profile-root>\IndexedDB" `
  "<claude-app-profile-root>\Local Storage\leveldb" `
  "<claude-app-profile-root>\Session Storage"
```

Use line previews only when needed:

```powershell
rg -a -n --hidden --no-messages "<sentinel>" "<specific-file-or-small-directory>"
```

If `rg` is unavailable, use platform equivalents, but keep the same order: transcript JSONL first, session metadata second, Chromium state last.

## Reading JSONL Safely

Use targeted extraction instead of dumping whole files.

On macOS/Linux, inspect `audit.jsonl` event type, timestamp, role, and short text excerpts:

```bash
jq -r '
  [.type,
   (._audit_timestamp // .timestamp // ""),
   (.message.role // ""),
   (if (.message.content|type) == "string" then .message.content
    elif .message.content then ([.message.content[]? | select(.type=="text") | .text][0] // "")
    else (.result // "") end)
  ] | @tsv
' /path/to/audit.jsonl | sed -n '1,120p'
```

On Windows PowerShell, use `ConvertFrom-Json` for the same audit-log summary:

```powershell
Get-Content "<audit.jsonl>" |
  ForEach-Object {
    $o = $_ | ConvertFrom-Json
    [pscustomobject]@{
      type = $o.type
      timestamp = $o._audit_timestamp ?? $o.timestamp
      role = $o.message.role
      text = if ($o.message.content -is [string]) {
        $o.message.content
      } elseif ($o.message.content) {
        (($o.message.content | Where-Object type -eq 'text' | Select-Object -First 1).text)
      } else {
        $o.result
      }
    }
  }
```

For Claude Code transcript snippets:

```bash
jq -r '
  select(.type=="user" or .type=="assistant" or .type=="summary")
  | [.timestamp, .type, (.message.content // .summary // "" | tostring)] | @tsv
' /path/to/session.jsonl | sed -n '1,160p'
```

If the JSONL shape differs, list top-level event types first:

```bash
jq -r '.type // "missing"' /path/to/session.jsonl | sort | uniq -c
```

## Summarize Evidence

When using old sessions for a writeup, capture:

- `Surface`: Claude Code, Claude Desktop, Cowork, or mixed.
- `Source`: path class and exact file path when useful.
- `Question`: what the user or agent was investigating.
- `Finding`: the shortest defensible conclusion.
- `Evidence`: sentinel match, timestamp, title, model, command output, or short safe excerpt.

Prefer a small Markdown evidence note in the current repo over relying on memory. Redact account/workspace IDs and private usernames unless the user explicitly needs exact paths.

## Final Answer Shape

Keep the answer concise and privacy-aware.

Include:

- The Claude surface found: Claude Code, Claude Desktop, Cowork, or mixed.
- The durable transcript location.
- The metadata location.
- Any draft/cache location if relevant.
- What the sentinel proved.
- A note that full local history may be incomplete if conversations are cloud-only or not cached locally.

Avoid:

- Raw tokens, cookies, credentials, HMAC keys, or OAuth material.
- Full account IDs or organization IDs unless required.
- Large transcript excerpts.
- Machine-specific usernames when an environment-variable path is enough.

## Common Pitfalls

- Do not assume `~/.claude/history.jsonl` is the full transcript.
- Do not assume Desktop, Code, and Cowork share the same store.
- Do not print raw `systemPrompt`, cookies, credentials, HMAC keys, or account identifiers.
- Do not recurse through all of `~/Library` or `%LOCALAPPDATA%` first; search known roots.
- Do not treat Chromium storage as canonical unless no transcript or metadata file contains the evidence.
- Do not edit files discovered in history unless the current user request requires it.
