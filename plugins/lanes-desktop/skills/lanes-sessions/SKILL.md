---
name: lanes-sessions
description: Use when managing Lanes issues or driving Claude Code sessions through the lanes_* MCP tools — creating issues, starting/stopping/inspecting sessions, batch-launching work across worktrees, reading terminal output, attaching labels and components by UUID, or moving issues across the backlog/todo/in-progress/done columns. Skill applies whenever a request mentions "Lanes", "lanes board", "lanes issue", "lanes session", or any lanes_* tool name.
---

# Lanes sessions

Lanes is a desktop app that puts every AI coding session on an issue board. It exposes a local MCP server (SSE on `http://localhost:5353/sse`) with a set of `lanes_`-prefixed tools for issue CRUD, session orchestration, history, and metadata lookups.

A single issue can host **multiple concurrent CLI sessions** (Claude, Codex, shell). Session-targeting tools take an optional `session` ref (UUID, auto-assigned slot, or name) to pick which one — see the "Multi-session model" section below.

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
| | `lanes_stop_session` | Stop a session for `issueId`. Optional `session` (UUID/slot/name) to disambiguate when >1. |
| | `lanes_resume_session` | Re-attach Claude to a stopped session. **Claude-only.** Optional `session`. |
| | `lanes_delete_session` | Permanently delete a session (stops it first if running). Optional `session`. |
| | `lanes_get_session_status` | With `issueId`: bare array of every session for that issue. Without: envelope `{ sessions, appliedFilters, truncated, totalAvailable }` capped at 20. |
| History | `lanes_get_issue_changes` | `git diff` for the issue's cwd, by `id`. |
| | `lanes_get_issue_history` | Paginated Claude conversation history, by `id`. Use `cliSessionId` to pick when an issue has multiple Claude sessions. |
| | `lanes_read_terminal` | Last `lines` (default 200, max 2000) of PTY scrollback. Optional `session`. |
| | `lanes_get_session_stats` | Tokens, model breakdown, tool calls, duration, by `id`. Use `cliSessionId` to pick when an issue has multiple Claude sessions. |
| Metadata | `lanes_list_labels` | UUIDs, names, colors. Call before assigning `tags`. |
| | `lanes_list_components` | UUIDs, names, project IDs. Call before setting `componentId`. |
| | `lanes_delete_worktree` | Remove a git worktree on disk by `projectPath` + `name`. |

Note the parameter-name inconsistency: issue endpoints take `id`, session endpoints take `issueId`. Don't mix them. Session-targeting tools additionally accept an optional `session` ref (UUID, slot, or name — case-insensitive).

## Multi-session model

An issue can host any number of concurrent CLI sessions. Each session is addressable three ways:

- **`session_uuid`** — Lanes' internal stable ID.
- **`slot`** — auto-assigned ordinal at spawn (`"1"`, `"2"`, …, lowest free integer).
- **`name`** — optional user-editable label.

Session-aware tools (`lanes_stop_session`, `lanes_resume_session`, `lanes_delete_session`, `lanes_read_terminal`) take `session: string` resolved in that order, case-insensitive. When `session` is omitted:

- **One session for the issue** → tool proceeds against it. Happy path is unchanged.
- **Multiple sessions** → tool returns an **MCP success result** (not an error) whose text starts `"Issue N has multiple sessions…"` and whose `_meta.sessions` array lists each candidate as `{ sessionUuid, slot, name, runtimeStatus, cli, cliSessionId, createdAt }`. Pick one (ask the user if not obvious) and retry with `session` set to its slot, name, or UUID.

`lanes_resume_session` is the **only** way to re-attach a stopped Claude session. Do **not** pass `--resume`, `--continue`, `--session-id`, or `--fork-session` to `lanes_start_session` — those flags are hard-rejected. Codex and shell sessions have no resume semantics; start fresh.

`cliSessionId` (the CLI's own resume token — for Claude, the `--resume <uuid>` value) is distinct from `session_uuid` (the Lanes ref used in `session`). `lanes_get_issue_history` and `lanes_get_session_stats` disambiguate via `cliSessionId`; everything else uses `session`. Don't conflate them.

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

Multi-session issues: step 1 returns the candidate array — pick one and pass `session` to step 2 (and `cliSessionId` to steps 3/4) to scope subsequent reads.

If a Claude session has stopped and you want to pick up where it left off, call `lanes_resume_session { issueId, session? }` — see the next workflow.

### Resume a stopped Claude session

```
1. lanes_get_session_status { issueId }                                   // see what's there
2. lanes_resume_session { issueId, session: "<slot|name|uuid>" }          // session optional if only one
3. lanes_read_terminal { issueId, session: "<same>" }                     // verify it's live again
```

Only works for `cli: "claude"` sessions that recorded a `cliSessionId`. Codex/shell sessions can't be resumed — use `lanes_start_session` to launch a fresh one.

### Mark complete

```
1. lanes_get_issue_changes { id }     // confirm there is real work to land
2. lanes_stop_session { issueId, session? }   // pass session on multi-session issues
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
- ❌ Passing `--resume`, `--continue`, `--session-id`, or `--fork-session` to `lanes_start_session`. These are rejected — use `lanes_resume_session` to re-attach a stopped Claude session.
- ❌ Treating the ambiguity response from `lanes_stop_session` / `lanes_resume_session` / `lanes_delete_session` / `lanes_read_terminal` as an error. It's a normal MCP success result with `_meta.sessions` — pick a candidate and retry with `session` set.
- ❌ Confusing `cliSessionId` with `session`. `cliSessionId` is the CLI's own resume token (only consumed by `lanes_get_issue_history` and `lanes_get_session_stats`); `session` is the Lanes ref (UUID / slot / name) used by stop / resume / delete / read_terminal.
