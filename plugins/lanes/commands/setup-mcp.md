---
description: Connect Claude Code to the running Lanes app via SSE MCP at localhost:5353
allowed-tools: Bash
---

Set up the Lanes MCP server in Claude Code. Steps:

1. **Check current state.** Run `claude mcp list` and look for an entry named `lanes`.
2. **If `lanes` is missing**, register it:
   ```
   claude mcp add --transport sse lanes http://localhost:5353/sse --scope user
   ```
   `--scope user` writes to `~/.claude.json` so it applies across all your projects.
3. **Verify registration** with `claude mcp list` — confirm `lanes` shows up with the SSE URL.
4. **Probe the endpoint** to confirm Lanes is actually running:
   ```
   curl -sS -m 2 -o /dev/null -w "%{http_code}\n" http://localhost:5353/sse
   ```
   - `200` → Lanes is up; the MCP is reachable.
   - Connection refused / timeout → Lanes isn't running. Tell the user to launch the Lanes desktop app (`open -a Lanes` on macOS), then re-run this command.
5. **Report.** One short paragraph: whether `lanes` is registered, whether the endpoint is live, and what the user should do next (e.g. "open Claude Code in a fresh terminal so the new MCP loads", or "launch Lanes").

Notes:
- Don't overwrite an existing `lanes` MCP entry without asking — the user may have customised the URL.
- The SSE endpoint is local-only (no auth, never leaves the machine). Don't expose `localhost:5353` over the network.
- If the user is on a non-default port (set in Lanes Settings → Local MCP), they should pass that URL instead.
