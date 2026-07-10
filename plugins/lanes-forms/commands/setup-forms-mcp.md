---
description: Connect Claude Code and Cursor to the Lanes Forms MCP (hosted at api.lanes.sh/mcp) so an agent can provision and fill forms
allowed-tools: Bash
---

Set up the `lanes-forms` MCP server in every installed agent client. Run the section for each client detected; skip sections for clients that aren't installed. Both clients can coexist, and this command is idempotent.

The MCP exposes five tools — `create_form`, `get_form`, `update_form`, `generate_form_snippet`, `submit_form`. The default (and easiest) transport is the **hosted** server at `https://api.lanes.sh/mcp`, which needs no local toolchain — only a **workspace API key** (`lfk_...`) as a Bearer credential.

## 1. Get the workspace key

The hosted MCP authenticates every request with a workspace key.

- If `FORMS_API_KEY` is already set in the environment, use it: `printenv FORMS_API_KEY` (starts with `lfk_`).
- Otherwise ask the user to paste their `lfk_...` workspace key. If they don't have one, tell them to mint one from the Lanes dashboard's **API keys** page (admin role), or via the API: `GET /v1/me` for the `workspace_id`, then `POST /v1/workspaces/{workspace_id}/api-keys` with `{"name":"cli"}` — the `key` is shown once.

Hold the key in a shell variable for the commands below (never echo it into logs): `KEY="lfk_..."`. Do not proceed without a key — the hosted MCP returns `401` unauthenticated. (Anonymous, keyless provisioning is gated off by default, so a key is required to actually create forms today.)

## 2. Detect installed clients

- Claude Code: `command -v claude` (non-empty path means installed).
- Cursor: `test -d ~/.cursor || test -d /Applications/Cursor.app` (either is sufficient).

If neither is installed, stop and tell the user.

## 3. Claude Code

Skip if Claude Code wasn't detected.

1. **Check current state.** Run `claude mcp list` and look for an entry named `lanes-forms`. If it's already present, don't overwrite it without asking (the user may have set a custom URL or key).
2. **If `lanes-forms` is missing**, register the hosted server (`--scope user` writes to `~/.claude.json` so it applies across all projects):
   ```
   claude mcp add --transport http lanes-forms https://api.lanes.sh/mcp \
     --header "Authorization: Bearer $KEY" --scope user
   ```
3. **Verify** with `claude mcp list`; confirm `lanes-forms` shows up.

## 4. Cursor

Skip if Cursor wasn't detected. Cursor has no `cursor mcp add` CLI, so edit `~/.cursor/mcp.json` directly (using `jq` to preserve any other `mcpServers` entries).

1. **Check current state.** `jq -r '.mcpServers["lanes-forms"] // empty' ~/.cursor/mcp.json 2>/dev/null` — non-empty means it's already registered (don't overwrite without asking).
2. **If missing**, register it (preferred, uses `jq`):
   ```
   mkdir -p ~/.cursor
   [ -f ~/.cursor/mcp.json ] || echo '{}' > ~/.cursor/mcp.json
   tmp=$(mktemp)
   jq --arg key "$KEY" '.mcpServers["lanes-forms"] = {"url":"https://api.lanes.sh/mcp","headers":{"Authorization":("Bearer " + $key)}}' \
      ~/.cursor/mcp.json > "$tmp" && mv "$tmp" ~/.cursor/mcp.json
   ```
   Fallback if `jq` isn't installed:
   ```
   KEY="$KEY" python3 - <<'PY'
   import json, os, pathlib
   p = pathlib.Path.home() / '.cursor' / 'mcp.json'
   p.parent.mkdir(exist_ok=True)
   data = json.loads(p.read_text()) if p.exists() else {}
   data.setdefault('mcpServers', {})['lanes-forms'] = {
       'url': 'https://api.lanes.sh/mcp',
       'headers': {'Authorization': 'Bearer ' + os.environ['KEY']},
   }
   p.write_text(json.dumps(data, indent=2) + '\n')
   PY
   ```
3. **Verify** with `jq '.mcpServers["lanes-forms"].url' ~/.cursor/mcp.json`.

## 5. Probe the endpoint

Run once, regardless of which clients were set up:
```
curl -sS -m 5 -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $KEY" https://api.lanes.sh/mcp
```
- `200` (or an event-stream response) → the MCP is reachable and the key is accepted.
- `401` → the key is missing, wrong, or revoked. Re-check step 1.
- Connection error / timeout → the API is unreachable; check the network and `https://api.lanes.sh` status.

## 6. Report

One short paragraph: per-client registration status (Claude Code, Cursor), whether the probe passed, and the restart hint — Claude Code needs a fresh terminal so the new MCP loads; Cursor needs the IDE (or `cursor-agent`) restarted. Then confirm the user can ask the agent to "provision a contact form" to test it.

## Notes

- The workspace key is a **secret**. It's stored in `~/.claude.json` / `~/.cursor/mcp.json`; don't print it back to the user or paste it into shared logs.
- **Local development** against a running `api/` checkout instead of production: register the stdio server from the local package (no PyPI needed):
  ```
  claude mcp add lanes-forms \
    -e FORMS_API_URL=http://localhost:8080 -e FORMS_API_KEY="$KEY" \
    -- uv --directory /path/to/lanes/api/packages/forms-mcp run lanes-forms-mcp
  ```
- Once `lanes-forms-mcp` is published to PyPI, the toolchain-free stdio alternative is `claude mcp add lanes-forms -e FORMS_API_KEY="$KEY" -- uvx lanes-forms-mcp`. Until then, use the hosted transport above.
