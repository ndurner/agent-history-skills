---
name: claude-history
description: Locate and interpret local conversation history for Claude Desktop, Claude Code, and Claude Cowork. Use when the user asks where Claude stores chat transcripts, Claude Code JSONL sessions, Cowork audit logs, local drafts, IndexedDB/LevelDB state, or Claude conversation history; when searching Claude history; or when preparing a privacy-safe report of where Claude conversations live locally. As Claude can emerge without upfront notice, verifying this guidance is encouraged.
---

# Claude History

## Operating Rules

Work read-only unless the user explicitly asks for export, cleanup, or migration.

Treat transcripts, logs, local app state, cookies, credentials, OAuth tokens, account IDs, and organization IDs as sensitive. Do not expose secrets or unnecessary machine-specific details in the final answer.

Report paths with environment-variable forms such as `%USERPROFILE%`, `%APPDATA%`, `%LOCALAPPDATA%`, `$HOME`, or `~` where possible.

Prefer a sentinel search supplied by the user. Search for exact text first. Use case-insensitive partials only if exact search fails.

Quote conversation content only when needed to prove that a location is correct, and quote the minimum possible excerpt.

## Core Model

Claude conversation history may exist in more than one local store.

Claude Code CLI history is usually filesystem-native: JSONL transcripts under a home-directory `.claude` tree.

Claude Desktop and Cowork history is usually app-profile-native: session metadata, audit logs, app logs, and Chromium storage under a desktop app profile. On Windows, Store-packaged apps may store this profile under `%LOCALAPPDATA%\Packages\<package>\LocalCache\Roaming\<AppName>`.

Chromium storage can contain useful prompt fragments or drafts even when a regular text search misses them. Search `IndexedDB`, `Local Storage\leveldb`, and `Session Storage` with binary-safe search.

## Workflow

1. Identify installed Claude surfaces:
   - Look for CLI binaries such as `claude`.
   - Look for `%USERPROFILE%\.claude` or `~/.claude`.
   - Look for Claude app profiles under `%APPDATA%`, `%LOCALAPPDATA%`, and `%LOCALAPPDATA%\Packages`.

2. Search durable transcript stores first:
   - `history.jsonl`
   - `projects/**/<session-id>.jsonl`
   - `projects/**/<session-id>/subagents/*.jsonl`
   - `local-agent-mode-sessions/**/audit.jsonl`
   - `local-agent-mode-sessions/**/local_*.json`
   - `claude-code-sessions/**/local_*.json`

3. Search app state second:
   - `logs`
   - `IndexedDB`
   - `IndexedDB\*.blob`
   - `Local Storage\leveldb`
   - `Session Storage`

4. Map metadata to transcripts:
   - `local_*.json` files often contain session title, initial message, model, created timestamp, workspace/cwd, and a local session id.
   - A sibling directory with the same `local_*` id often contains `audit.jsonl`, nested `.claude` runtime state, output files, and temporary files.
   - `history.jsonl` is usually a lightweight prompt index. The full Claude Code transcript is usually in `projects/.../*.jsonl`.

5. Summarize findings:
   - Separate durable transcripts, session metadata, UI drafts/cache, and logs.
   - Say whether the sentinel was found in transcript content, session metadata, UI draft/cache, or logs.
   - Mention residual uncertainty, especially for cloud-only conversations that are not locally cached.

## Common Claude Locations

Use these as patterns and confirm locally.

### Claude Code CLI

Likely locations:

```text
~/.claude/history.jsonl
~/.claude/projects/<encoded-working-directory>/<session-id>.jsonl
~/.claude/projects/<encoded-working-directory>/<session-id>/subagents/*.jsonl
~/.claude/sessions/*.json
```

On Windows, `~` is usually `%USERPROFILE%`.

### Claude Desktop / Cowork

Likely Windows profile roots:

```text
%APPDATA%\Claude
%LOCALAPPDATA%\Claude
%LOCALAPPDATA%\Packages\<ClaudePackage>\LocalCache\Roaming\Claude
```

Within the profile root, check:

```text
local-agent-mode-sessions\<account-or-user-id>\<org-or-workspace-id>\local_*.json
local-agent-mode-sessions\<account-or-user-id>\<org-or-workspace-id>\local_*\
local-agent-mode-sessions\<account-or-user-id>\<org-or-workspace-id>\local_*\audit.jsonl
claude-code-sessions\<account-or-user-id>\<org-or-workspace-id>\local_*.json
IndexedDB\*.leveldb
IndexedDB\*.blob
Local Storage\leveldb
Session Storage
logs
```

Do not report raw account or organization IDs unless the user explicitly needs them.

## Search Approach

Start with candidate roots. On Windows, useful discovery commands include:

```powershell
Get-ChildItem -Force $env:USERPROFILE, $env:APPDATA, $env:LOCALAPPDATA |
  Where-Object { $_.Name -match 'claude|anthropic' }

Get-ChildItem -Force "$env:LOCALAPPDATA\Packages" -Directory |
  Where-Object { $_.Name -match 'claude|anthropic' }
```

Search text transcripts first:

```powershell
rg -n --hidden --no-messages -i "<sentinel>" "$env:USERPROFILE\.claude"
```

Search Desktop/Cowork app profiles next. Prefer narrow roots over all of `%LOCALAPPDATA%`:

```powershell
rg -n --hidden --no-messages -i "<sentinel>" "<claude-app-profile-root>"
```

Use binary-safe search for Chromium stores:

```powershell
rg -a --files-with-matches --hidden --no-messages -i "<sentinel>" `
  "<claude-app-profile-root>\IndexedDB" `
  "<claude-app-profile-root>\Local Storage\leveldb" `
  "<claude-app-profile-root>\Session Storage"
```

Use line previews only when needed:

```powershell
rg -a -n --hidden --no-messages -i "<sentinel>" "<specific-file-or-small-directory>"
```

If `rg` is unavailable, use platform equivalents, but keep the same order: transcript JSONL first, session metadata second, Chromium state last.

## Reading JSONL Safely

Claude transcript JSONL often contains mixed event types. Extract summaries rather than dumping entire files.

For a small `audit.jsonl`, inspect event type, timestamp, role, and short text excerpts:

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

For large files, filter with `rg` first and then parse only the relevant lines.

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

- Raw tokens, cookies, credentials, or keys.
- Full account IDs or organization IDs unless required.
- Large transcript excerpts.
- Machine-specific usernames when an environment-variable path is enough.
