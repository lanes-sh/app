---
description: Connect Claude Code and Cursor to the Lanes remote MCP (hosted at api.lanes.sh/mcp) so an agent can provision and fill forms
allowed-tools: Bash
---

Set up the `lanes` MCP server in every installed agent client. Run the section for each client detected; skip sections for clients that aren't installed. Both clients can coexist, and this command is idempotent.

The MCP exposes five tools: `create_form`, `get_form`, `update_form`, `generate_form_snippet`, `submit_form`. It's hosted over streamable HTTP at `https://api.lanes.sh/mcp` and needs no local toolchain. No signup or key is required to start: `create_form` and `submit_form` work anonymously. A **workspace API key** (`lfk_...`), passed as a Bearer credential, is only needed to unlock the management tools (`get_form`, `update_form`, `generate_form_snippet`) and to make new forms born claimed into your workspace.

## 1. Optional: a workspace key for the management tools

Skip this step to register the server anonymously (the default). Add a key only if you want the management tools or born-claimed provisioning.

- If `LANES_API_KEY` is already set in the environment, use it: `printenv LANES_API_KEY` (starts with `lfk_`).
- Otherwise ask the user to paste their `lfk_...` workspace key. If they don't have one, they can mint one from the Lanes dashboard's **API keys** page (admin role), or via the API: `GET /v1/me` for the `workspace_id`, then `POST /v1/workspaces/{workspace_id}/api-keys` with `{"name":"cli"}` (the `key` is shown once).

If you have a key, hold it in a shell variable for the commands below (never echo it into logs): `KEY="lfk_..."`. No key is fine: register without the header and the anonymous tools still work.

## 2. Detect installed clients

- Claude Code: `command -v claude` (non-empty path means installed).
- Cursor: `test -d ~/.cursor || test -d /Applications/Cursor.app` (either is sufficient).

If neither is installed, stop and tell the user.

## 3. Claude Code

Skip if Claude Code wasn't detected.

1. **Check current state.** Run `claude mcp list` and look for an entry named `lanes`. If it's already present, don't overwrite it without asking (the user may have set a custom URL or key).
2. **If `lanes` is missing**, register the hosted server (`--scope user` writes to `~/.claude.json` so it applies across all projects). Anonymous (default):
   ```
   claude mcp add --transport http lanes https://api.lanes.sh/mcp --scope user
   ```
   If you have a workspace key from step 1, add it to unlock the management tools:
   ```
   claude mcp add --transport http lanes https://api.lanes.sh/mcp \
     --header "Authorization: Bearer $KEY" --scope user
   ```
3. **Verify** with `claude mcp list`; confirm `lanes` shows up.

## 4. Cursor

Skip if Cursor wasn't detected. Cursor has no `cursor mcp add` CLI, so edit `~/.cursor/mcp.json` directly (using `jq` to preserve any other `mcpServers` entries).

1. **Check current state.** `jq -r '.mcpServers["lanes"] // empty' ~/.cursor/mcp.json 2>/dev/null` (non-empty means it's already registered; don't overwrite without asking).
2. **If missing**, register it. Anonymous (default, uses `jq`):
   ```
   mkdir -p ~/.cursor
   [ -f ~/.cursor/mcp.json ] || echo '{}' > ~/.cursor/mcp.json
   tmp=$(mktemp)
   jq '.mcpServers["lanes"] = {"url":"https://api.lanes.sh/mcp"}' \
      ~/.cursor/mcp.json > "$tmp" && mv "$tmp" ~/.cursor/mcp.json
   ```
   With a workspace key (adds the Bearer header):
   ```
   tmp=$(mktemp)
   jq --arg key "$KEY" '.mcpServers["lanes"] = {"url":"https://api.lanes.sh/mcp","headers":{"Authorization":("Bearer " + $key)}}' \
      ~/.cursor/mcp.json > "$tmp" && mv "$tmp" ~/.cursor/mcp.json
   ```
   Fallback if `jq` isn't installed (omits the header when no key is set):
   ```
   KEY="${KEY:-}" python3 - <<'PY'
   import json, os, pathlib
   p = pathlib.Path.home() / '.cursor' / 'mcp.json'
   p.parent.mkdir(exist_ok=True)
   data = json.loads(p.read_text()) if p.exists() else {}
   server = {'url': 'https://api.lanes.sh/mcp'}
   key = os.environ.get('KEY', '')
   if key:
       server['headers'] = {'Authorization': 'Bearer ' + key}
   data.setdefault('mcpServers', {})['lanes'] = server
   p.write_text(json.dumps(data, indent=2) + '\n')
   PY
   ```
3. **Verify** with `jq '.mcpServers["lanes"].url' ~/.cursor/mcp.json`.

## 5. Probe the endpoint

Run once, regardless of which clients were set up (the `${KEY:+...}` adds the header only if a key is set):
```
curl -sS -m 5 -o /dev/null -w "%{http_code}\n" \
  ${KEY:+-H "Authorization: Bearer $KEY"} https://api.lanes.sh/mcp
```
- `200` (or an event-stream response) → the MCP is reachable; if you passed a key, it was accepted.
- `401` with a key → the key is wrong or revoked. Re-check step 1. (Without a key, the anonymous tools still work.)
- Connection error / timeout → the API is unreachable; check the network and `https://api.lanes.sh` status.

## 6. Report

One short paragraph: per-client registration status (Claude Code, Cursor), whether a key was configured (or the server was registered anonymously), whether the probe passed, and the restart hint. Claude Code needs a fresh terminal so the new MCP loads; Cursor needs the IDE (or `cursor-agent`) restarted. Then confirm the user can ask the agent to "provision a contact form" to test it.

## Notes

- The workspace key is a **secret** and is optional. When set it's stored in `~/.claude.json` / `~/.cursor/mcp.json`; don't print it back to the user or paste it into shared logs.
- **Local development** against a running `api/` checkout instead of production: register the stdio server from the local package (no PyPI needed):
  ```
  claude mcp add lanes \
    -e LANES_API_URL=http://localhost:8080 -e LANES_API_KEY="$KEY" \
    -- uv --directory /path/to/lanes/api/packages/lanes-mcp run lanes-mcp
  ```
  The hosted HTTP transport above needs no local package, so most users won't need this. This `LANES_API_URL=http://localhost:8080` override is **only** for Lanes maintainers developing the `api/` service itself. It does not change where a provisioned form is submitted from: a form you wire into a real site must always post to its production endpoint `https://api.lanes.sh/v1/f/{id}`, never a local base (`localhost` is an allowed submit origin, so prod works from local dev too).
