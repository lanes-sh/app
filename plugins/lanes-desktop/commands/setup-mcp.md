---
description: Connect Claude Code and Cursor to the running Lanes app via SSE MCP at localhost:5353
allowed-tools: Bash
---

Set up the Lanes MCP server in every installed agent client. Run the section for each client detected; skip sections for clients that aren't installed. Both clients can coexist, and this command is idempotent.

## 1. Detect installed clients

- Claude Code: `command -v claude` (non-empty path means installed).
- Cursor: `test -d ~/.cursor || test -d /Applications/Cursor.app` (either is sufficient).

If neither is installed, stop and tell the user.

## 2. Claude Code

Skip if Claude Code wasn't detected.

1. **Check current state.** Run `claude mcp list` and look for any of three names, since the local server has shipped under all of them:
   - `lanes-desktop` — the current name.
   - `lanes-local` — the previous name. Lanes renames this itself the next time the desktop app launches, so leave it be.
   - `lanes` whose URL is `http://localhost:5353/sse` — the original name. Still supported; the user can rename it from Lanes Settings → Local MCP.

   Any of the three means the local server is already registered, so skip to step 3 and don't add a duplicate. (A `lanes` entry pointing somewhere *else* is the separate remote Lanes Forms server; leave it alone.)
2. **If none are present**, register it:
   ```
   claude mcp add --transport sse lanes-desktop http://localhost:5353/sse --scope user
   ```
   `--scope user` writes to `~/.claude.json` so it applies across all your projects.
3. **Verify registration** with `claude mcp list`. Confirm `lanes-desktop` (or whichever older name you already had) shows up with the SSE URL.

## 3. Cursor

Skip if Cursor wasn't detected. Cursor has no `cursor mcp add` CLI, so we edit `~/.cursor/mcp.json` directly (using `jq` to preserve any other `mcpServers` entries the user has).

Unlike Claude Code and Codex, Cursor's config is never written by the Lanes app, so this command is the only thing that migrates an older entry name here.

1. **Check current state.** Run:
   ```
   jq -r '(.mcpServers["lanes-desktop"] // .mcpServers["lanes-local"] // .mcpServers.lanes).url // empty | select(contains("5353"))' ~/.cursor/mcp.json 2>/dev/null
   ```
   Non-empty output means the local server is already registered, under the current name `lanes-desktop` or under an older `lanes-local` / `lanes` still pointing at `localhost:5353`. A bare `lanes` entry pointing elsewhere is the separate remote Lanes Forms server; leave it alone and still add `lanes-desktop` below.
2. **If it is registered as `lanes-local`**, rename the key in place. This preserves whatever URL the user set, so it is safe even on a custom port:
   ```
   tmp=$(mktemp)
   jq '.mcpServers["lanes-desktop"] = .mcpServers["lanes-local"] | del(.mcpServers["lanes-local"])' \
      ~/.cursor/mcp.json > "$tmp" && mv "$tmp" ~/.cursor/mcp.json
   ```
   Leave a legacy bare `lanes` entry alone — renaming that one is the user's call, since the name is also used by the remote Lanes Forms server.
3. **If none are present**, register it. Preferred (uses `jq`):
   ```
   mkdir -p ~/.cursor
   [ -f ~/.cursor/mcp.json ] || echo '{}' > ~/.cursor/mcp.json
   tmp=$(mktemp)
   jq '.mcpServers["lanes-desktop"] = {"url": "http://localhost:5353/sse"}' \
      ~/.cursor/mcp.json > "$tmp" && mv "$tmp" ~/.cursor/mcp.json
   ```
   Fallback if `jq` isn't installed:
   ```
   python3 - <<'PY'
   import json, pathlib
   p = pathlib.Path.home() / '.cursor' / 'mcp.json'
   p.parent.mkdir(exist_ok=True)
   data = json.loads(p.read_text()) if p.exists() else {}
   servers = data.setdefault('mcpServers', {})
   # Rename an older entry rather than adding a second one, preserving its URL.
   if 'lanes-local' in servers:
       servers.setdefault('lanes-desktop', servers.pop('lanes-local'))
   else:
       servers.setdefault('lanes-desktop', {'url': 'http://localhost:5353/sse'})
   p.write_text(json.dumps(data, indent=2) + '\n')
   PY
   ```
4. **Verify** with `jq '.mcpServers["lanes-desktop"]' ~/.cursor/mcp.json` (or `python3 -c "import json,pathlib;print(json.loads(pathlib.Path.home().joinpath('.cursor/mcp.json').read_text())['mcpServers']['lanes-desktop'])"`). Confirm the URL entry is present.

## 4. Probe the endpoint

Run once, regardless of which clients were set up:
```
curl -sS -m 2 -o /dev/null -w "%{http_code}\n" http://localhost:5353/sse
```
- `200` → Lanes is up; the MCP is reachable.
- Connection refused / timeout → Lanes isn't running. Tell the user to launch the Lanes desktop app (`open -a Lanes` on macOS), then re-run this command.

## 5. Report

One short paragraph covering: per-client registration status (Claude Code, Cursor), whether the endpoint is live, and the restart hint. Claude Code needs a fresh terminal so the new MCP loads; Cursor needs the IDE (or `cursor-agent`) restarted.

Notes:
- New installs register the local server as `lanes-desktop`. It previously shipped as `lanes-local`, and before that as bare `lanes`; both keep working.
  - In Claude Code and Codex, the Lanes app renames `lanes-local` itself at launch, and moves the `mcp__lanes-local__*` tool permissions with it so nothing needs re-approving. In Cursor, this command does the rename (step 3.2) because the app never touches Cursor's config.
  - Bare `lanes` is never renamed automatically anywhere, because the remote Lanes Forms server uses that name. Point the user at Lanes Settings → Local MCP if they want to move it.
- Don't overwrite an existing MCP entry's *value* without asking. The user may have customised the URL. Renaming a key while preserving its value is fine, and is what step 3.2 does.
- The SSE endpoint is local-only (no auth, never leaves the machine). Don't expose `localhost:5353` over the network.
- If the user is on a non-default port (set in Lanes Settings → Local MCP), they should pass that URL instead, in both clients.
