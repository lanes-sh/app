---
name: lanes-sessions
description: Use when managing Lanes issues or driving Claude Code sessions through the lanes_* MCP tools — creating issues, starting/stopping/inspecting sessions, batch-launching work across worktrees, reading terminal output, attaching labels and components by UUID, or moving issues across the backlog/todo/in-progress/done columns. Skill applies whenever a request mentions "Lanes", "lanes board", "lanes issue", "lanes session", or any lanes_* tool name.
---

# Lanes sessions

Lanes is a desktop app that puts every AI coding session on an issue board. It exposes a local MCP server (SSE on `http://localhost:5353/sse`) with 15 tools — all prefixed `lanes_` — for issue CRUD, session orchestration, history, and metadata lookups.

Use this skill whenever the user wants to look at, create, or run work on the Lanes board through chat.

## Verify the MCP is connected

Before doing anything, sanity-check that the Lanes MCP is reachable:

- Call `lanes_list_components`. If it returns an array (possibly empty), you're good.
- If the tool isn't available at all, the MCP isn't installed. Tell the user to run the `/lanes:setup-mcp` slash command (shipped alongside this skill) or, if they have raw skills only, run:
  ```
  claude mcp add --transport sse lanes http://localhost:5353/sse --scope user
  ```
  Then restart Claude Code so the new MCP loads. Lanes itself must also be running (the desktop app must be open).

## Tool inventory

| Group | Tool | Purpose |
|---|---|---|
| Issues | `lanes_list_issues` | Filter by `step` / `tags` (any-match) / `componentId` / `search`. |
| | `lanes_get_issue` | Full details by `id`. |
| | `lanes_create_issue` | Required: `title`. |
| | `lanes_update_issue` | Patch by `id`; pass `null` to clear nullable fields. |
| | `lanes_delete_issue` | By `id`. Permanent. |
| | `lanes_move_issue` | Shorthand for `update_issue` with only `step`. |
| Sessions | `lanes_start_session` | Required: `issueId`. Spawns Claude Code (default), Codex, or `shell`. |
| | `lanes_stop_session` | Kills the PTY for `issueId`. |
| | `lanes_get_session_status` | All sessions, or filter to one `issueId`. |
| History | `lanes_get_issue_changes` | `git diff` for the issue's cwd, by `id`. |
| | `lanes_get_issue_history` | Paginated Claude conversation history, by `id`. |
| | `lanes_read_terminal` | Last `lines` (default 200, max 2000) of PTY scrollback for `issueId`. |
| | `lanes_get_session_stats` | Tokens, model breakdown, tool calls, duration, by `id`. |
| Metadata | `lanes_list_labels` | UUIDs, names, colors. Call before assigning `tags`. |
| | `lanes_list_components` | UUIDs, names, project IDs. Call before setting `componentId`. |

Note the parameter-name inconsistency: issue endpoints take `id`, session endpoints take `issueId`. Don't mix them.

## Critical gotchas

- **Labels and components are UUIDs, not names.** Always call `lanes_list_labels` / `lanes_list_components` first and resolve names → UUIDs locally. Passing a plain name like `"bug"` will not match anything.
- **`step` is exactly four values:** `backlog`, `todo`, `in-progress`, `done`. Any other string is rejected. Use `lanes_move_issue` if you only want to change the step.
- **Worktree auto-create requires two fields, set in advance.** To make `lanes_start_session` create a fresh git worktree, the issue must already have `worktreeStrategy: "create"` AND `worktreeName: "<branch>"`. Set them via `lanes_create_issue` or `lanes_update_issue` *before* calling `start_session`.
- **`prompt` has three modes:**
  - Omitted → uses the issue's description (falling back to title).
  - `""` (empty string) → starts the session with no prompt at all.
  - `"text"` → overrides the description for this session only.
- **`flags` array shape** for `lanes_start_session`:
  ```json
  [
    { "flag": "dangerously-skip-permissions", "value": "", "kind": "flag" },
    { "flag": "ANTHROPIC_MODEL", "value": "claude-opus-4-7", "kind": "env" }
  ]
  ```
  `kind: "flag"` becomes `--flag-name value`; `kind: "env"` becomes `ENV=value` prepended to the command.
- **`planMode: true`** only takes effect when `cli: "claude"`. It maps to `--permission-mode plan`.
- **`lanes_list_issues` filter combination:** filters AND together, except `tags`, which is any-match (issue must have ≥1 of the listed UUIDs).
- **`update_issue` to clear a field** requires explicitly passing `null` (or `[]` for `tags`/`dependencies`). Omitting a key leaves it untouched.

## Common workflows

### Create-and-launch a single issue

```
1. lanes_list_components                            // pick the right componentId
2. lanes_create_issue {
     title, description, cwd,
     componentId,
     worktreeStrategy: "create",
     worktreeName: "feat/short-slug"
   }
3. lanes_start_session { issueId, planMode: true }   // or omit planMode for implement-mode
4. lanes_get_session_status { issueId }              // confirm it actually started
```

### Batch-launch the backlog in plan mode

```
1. lanes_list_issues { step: "backlog" }
2. for each issue:
     lanes_start_session {
       issueId,
       planMode: true,
       flags: [{ flag: "dangerously-skip-permissions", value: "", kind: "flag" }]
     }
```

Then ask the user to review the resulting plans; promote each issue to `todo` or `in-progress` once plans are approved.

### Investigate a stuck session

```
1. lanes_get_session_status { issueId }     // is the PTY actually live?
2. lanes_read_terminal { issueId, lines: 500 }
3. lanes_get_issue_history { id: issueId, limit: 50 }   // structured Claude transcript
4. lanes_get_session_stats { id: issueId }              // tokens/tool calls/duration
```

If the terminal looks frozen, `lanes_stop_session` and `lanes_start_session` again — the issue's existing description is preserved.

### Mark complete

```
1. lanes_get_issue_changes { id }     // confirm there is real work to land
2. lanes_stop_session { issueId }
3. lanes_move_issue { id, step: "done" }
```

## Anti-patterns

- ❌ Calling `lanes_create_issue` with `tags: ["bug"]`. Tags must be label UUIDs from `lanes_list_labels`.
- ❌ Setting `step: "in_progress"` (underscore). It's `in-progress` (hyphen).
- ❌ Calling `lanes_start_session` with `worktreeStrategy: "create"` in the same call — the field lives on the *issue*, not the session call. Set it via `create_issue` / `update_issue` first.
- ❌ Polling `lanes_read_terminal` in a tight loop. Read once, summarise, ask the user before re-polling.
- ❌ Confusing `id` (issue endpoints) with `issueId` (session endpoints). They refer to the same thing but live under different keys.
- ❌ Using `lanes_update_issue` to "clear" a field by sending `""`. Empty string is a value; pass `null` to clear.
- ❌ Starting many sessions without `componentId` or `cwd` set. Sessions need a working directory; otherwise the PTY spawns wherever the Lanes app was launched from.
