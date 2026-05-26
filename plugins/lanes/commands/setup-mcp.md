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

1. **Check current state.** Run `claude mcp list` and look for an entry named `lanes`.
2. **If `lanes` is missing**, register it:
   ```
   claude mcp add --transport sse lanes http://localhost:5353/sse --scope user
   ```
   `--scope user` writes to `~/.claude.json` so it applies across all your projects.
3. **Verify registration** with `claude mcp list`. Confirm `lanes` shows up with the SSE URL.

## 3. Cursor

Skip if Cursor wasn't detected. Cursor has no `cursor mcp add` CLI, so we edit `~/.cursor/mcp.json` directly (using `jq` to preserve any other `mcpServers` entries the user has).

1. **Check current state.** Run:
   ```
   jq -r '.mcpServers.lanes // empty' ~/.cursor/mcp.json 2>/dev/null
   ```
   Non-empty output means `lanes` is already registered (don't overwrite without asking).
2. **If `lanes` is missing**, register it. Preferred (uses `jq`):
   ```
   mkdir -p ~/.cursor
   [ -f ~/.cursor/mcp.json ] || echo '{}' > ~/.cursor/mcp.json
   tmp=$(mktemp)
   jq '.mcpServers.lanes = {"url": "http://localhost:5353/sse"}' \
      ~/.cursor/mcp.json > "$tmp" && mv "$tmp" ~/.cursor/mcp.json
   ```
   Fallback if `jq` isn't installed:
   ```
   python3 - <<'PY'
   import json, pathlib
   p = pathlib.Path.home() / '.cursor' / 'mcp.json'
   p.parent.mkdir(exist_ok=True)
   data = json.loads(p.read_text()) if p.exists() else {}
   data.setdefault('mcpServers', {})['lanes'] = {'url': 'http://localhost:5353/sse'}
   p.write_text(json.dumps(data, indent=2) + '\n')
   PY
   ```
3. **Verify** with `jq '.mcpServers.lanes' ~/.cursor/mcp.json` (or `python3 -c "import json,pathlib;print(json.loads(pathlib.Path.home().joinpath('.cursor/mcp.json').read_text())['mcpServers']['lanes'])"`). Confirm the URL entry is present.

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
- Don't overwrite an existing `lanes` MCP entry without asking. The user may have customised the URL.
- The SSE endpoint is local-only (no auth, never leaves the machine). Don't expose `localhost:5353` over the network.
- If the user is on a non-default port (set in Lanes Settings → Local MCP), they should pass that URL instead, in both clients.
