---
name: lanes-sessions
description: Use when managing Lanes issues or driving Claude Code sessions through the lanes_* MCP tools — creating issues, starting/stopping/inspecting sessions, batch-launching work across worktrees, reading terminal output, attaching labels and components by UUID, or moving issues across the backlog/planning/implementation/review/done columns. Skill applies whenever a request mentions "Lanes", "lanes board", "lanes issue", "lanes session", or any lanes_* tool name. Also applies when Claude Code is itself running inside a Lanes session (`LANES_TERMINAL=1`, `LANES_SESSION=<issue id>`) — every worktree you create must be linked to the issue with lanes_update_issue, or the issue's Changes tab points at an empty diff instead of at your work.
---

# Lanes sessions

Lanes is a desktop app that puts every AI coding session on an issue board. It exposes a local MCP server (SSE on `http://localhost:5353/sse`) with a set of `lanes_`-prefixed tools for issue CRUD, session orchestration, history, and metadata lookups.

A single issue can host **multiple concurrent CLI sessions** (Claude, Codex, shell). Session-targeting tools take an optional `session` ref (UUID, auto-assigned slot, or name) to pick which one — see the "Multi-session model" section below.

Use this skill whenever the user wants to look at, create, or run work on the Lanes board through chat.

## Verify the MCP is connected

Before doing anything, sanity-check that the Lanes MCP is reachable:

- Call `lanes_list_components`. If it returns an array (possibly empty), you're good.
- If the tool isn't available at all, the MCP isn't installed. Tell the user to run the `/lanes-desktop:setup-mcp` slash command (shipped alongside this skill) or, if they have raw skills only, run:
  ```
  claude mcp add --transport sse lanes-desktop http://localhost:5353/sse --scope user
  ```
  Then restart Claude Code so the new MCP loads. Lanes itself must also be running (the desktop app must be open).

## Working inside a Lanes session

Everything else here assumes you are driving the board from outside. When Lanes launched *you*, three environment variables say so:

| Variable | Meaning |
|---|---|
| `LANES_TERMINAL` | `1` when Lanes spawned this shell. Unset means this section does not apply. |
| `LANES_SESSION` | The **issue ID** you are working on. This is what to pass as `id` / `issueId`. |
| `LANES_SESSION_UUID` | This session's Lanes ref, for the `session` param. |

Read them with `echo $LANES_SESSION`. If `LANES_TERMINAL` is unset you are an ordinary Claude Code session with no issue to report against, so skip to the tool inventory.

### Link every worktree you create

**Every `git worktree add` you run is followed by
`lanes_update_issue { id: $LANES_SESSION, branch: "<branch>" }`.** Not conditionally, not once you are finished. A worktree Lanes did not create is a worktree the issue does not know about, and until you link it the Changes tab and `lanes_get_issue_changes` both show an empty diff, so your work reads as never having happened.

Setting `branch` is the whole job. Lanes finds the worktree that branch is checked out in and fills `worktreeStrategy`, `worktreePath` and `worktreeName` from it, so do not set those three yourself.

Then check it took:

```
lanes_get_issue { id: $LANES_SESSION }
```

`worktreeStrategy` should be `"select"` and `worktreePath` should be the folder you are working in. If `worktreeStrategy` came back `"none"`, Lanes could not find a worktree for that branch. Call `lanes_list_worktrees`, find your folder in the list, and pass its `path`:

```
lanes_update_issue { id: $LANES_SESSION, worktreePath: "<path from the list>" }
```

If your folder is not in that list either, you are in a repository Lanes has no working folder linked for, and there is nothing to link the issue to. Say so rather than retrying.

When Lanes started you inside a worktree it made, `lanes_get_issue` already reports `"create"` or `"select"` and there is nothing to do. Checking costs one call and is worth it before you assume.

Linking is not the same as asking for a worktree. `worktreeStrategy: "create"` only affects the *next* session on the issue and never relocates a running agent, so setting it mid-session will not move you.

## Tool inventory

| Group | Tool | Purpose |
|---|---|---|
| Issues | `lanes_list_issues` | Filter by `step` / `tags` (any-match) / `componentId` / `search`. **Defaults are scoped**: at most 20, active project only, `done` excluded. Pass `limit` (no upper bound), `allProjects: true`, or `includeDone: true` to widen. Returns an envelope `{ issues, appliedFilters, truncated, totalAvailable }`, so check `truncated` before concluding a search found nothing. |
| | `lanes_get_issue` | Full details by `id`. Does **not** include sessions. |
| | `lanes_create_issue` | Required: `title`. |
| | `lanes_update_issue` | Patch by `id`; pass `null` to clear nullable fields. |
| | `lanes_delete_issue` | By `id`. Permanent. |
| | `lanes_move_issue` | Shorthand for `update_issue` with only `step`. |
| Sessions | `lanes_start_session` | Required: `issueId`. Spawns Claude Code (default), Codex, or `shell`. **Always adds a new session**; returns its slot + UUID. |
| | `lanes_stop_session` | Stop a session for `issueId`. Optional `session` (UUID/slot/name) to disambiguate when >1. |
| | `lanes_resume_session` | Re-attach Claude to a stopped session. **Claude-only.** Optional `session`. |
| | `lanes_delete_session` | Permanently delete a session (stops it first if running). Optional `session`. |
| | `lanes_get_session_status` | With `issueId`: bare array of every session for that issue (status under both `status` and `runtimeStatus`). Without: envelope `{ sessions, appliedFilters, truncated, totalAvailable }` capped at 20. |
| History | `lanes_get_issue_changes` | `git diff` for the issue's cwd, by `id`. |
| | `lanes_get_issue_history` | Paginated Claude conversation history, by `id`. Use `cliSessionId` to pick when an issue has multiple Claude sessions. |
| | `lanes_read_terminal` | Last `lines` (default 200, max 2000) of PTY scrollback. Optional `session`. |
| | `lanes_get_session_stats` | Tokens, model breakdown, tool calls, duration, by `id`. Use `cliSessionId` to pick when an issue has multiple Claude sessions. |
| Metadata | `lanes_list_labels` | UUIDs, names, colors. Call before assigning `tags`. |
| | `lanes_list_components` | UUIDs, names, project IDs. Call before setting `componentId`. |
| | `lanes_list_worktrees` | Every worktree Lanes can see: `name`, `path`, `branch`, and the `projectPath` it came from. Optional `projectPath` narrows it to one repository. Use it to find the path to link, or to check whether a branch already has a worktree. |
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

`lanes_get_session_status` is the read side and takes **no** `session` param. With `issueId` it returns a bare array of *every* session for that issue; each entry carries the runtime status under both `status` and `runtimeStatus` (same value, two keys). Filter that array yourself by slot / UUID.

`lanes_start_session` **always creates a new session.** It is never a way to re-attach or to "make sure" one is running — each call adds another slot. To re-attach, use `lanes_resume_session`.

`lanes_resume_session` is the **only** way to re-attach a stopped Claude session. Do **not** pass `--resume`, `--continue`, `--session-id`, or `--fork-session` to `lanes_start_session` — those flags are hard-rejected. Codex and shell sessions have no resume semantics; start fresh.

### Starting is asynchronous

`lanes_start_session` hands the launch to the Lanes app and waits up to ~10s for the new session to register, then returns its slot and UUID. Two things follow:

- **A response that says it could not confirm is not a failure.** The launch may still be in flight — worktree creation runs a real `git worktree add`, and if the issue has no usable cwd Lanes opens a folder picker and waits on the user indefinitely. Read `lanes_get_session_status { issueId }`, or ask the user to look at the Lanes window.
- **An empty session list is not proof the start failed.** In the window before the session registers, `lanes_get_session_status` returns `[]` and the session-aware tools return `"No sessions exist for issue N yet"`. Both are expected. Wait and re-read.

Never re-issue `lanes_start_session` to check on a previous one. That is how you end up with five identical sessions on one issue.

`cliSessionId` (the CLI's own resume token — for Claude, the `--resume <uuid>` value) is distinct from `session_uuid` (the Lanes ref used in `session`). `lanes_get_issue_history` and `lanes_get_session_stats` disambiguate via `cliSessionId`; everything else uses `session`. Don't conflate them.

## Critical gotchas

- **Labels and components are UUIDs, not names.** Always call `lanes_list_labels` / `lanes_list_components` first and resolve names → UUIDs locally. Passing a plain name like `"bug"` will not match anything.
- **`step` is one of six values:** `backlog`, `planning`, `implementation`, `review`, `done`, `misc`. The tool declares them as an enum, so anything else is rejected before it reaches the board. Use `lanes_move_issue` if you only want to change the step.
- **Worktree auto-create requires two fields, set in advance.** To make `lanes_start_session` create a fresh git worktree, the issue must already have `worktreeStrategy: "create"` AND `worktreeName: "<branch>"`. Set them via `lanes_create_issue` or `lanes_update_issue` *before* calling `start_session`. To attach an issue to a worktree that already exists (one you made yourself, mid-session), set `branch` instead — see "Link a worktree you created yourself".
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
                                                    // → returns the new slot + sessionUuid
```

Step 3 already tells you the slot it started; there is no separate confirmation step. If you do want to double-check, read `lanes_get_session_status { issueId }` and look for **that** slot — do not call `lanes_start_session` again.

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

Then ask the user to review the resulting plans; promote each issue to `implementation` once plans are approved.

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
- ❌ Setting `step: "todo"` or `"in-progress"`. Neither exists. The board's steps are `backlog`, `planning`, `implementation`, `review`, `done` and `misc`.
- ❌ Calling `lanes_start_session` with `worktreeStrategy: "create"` in the same call — the field lives on the *issue*, not the session call. Set it via `create_issue` / `update_issue` first.
- ❌ Polling `lanes_read_terminal` in a tight loop. Read once, summarise, ask the user before re-polling.
- ❌ Confusing `id` (issue endpoints) with `issueId` (session endpoints). They refer to the same thing but live under different keys.
- ❌ Using `lanes_update_issue` to "clear" a field by sending `""`. Empty string is a value; pass `null` to clear.
- ❌ Starting many sessions without `componentId` or `cwd` set. Sessions need a working directory; otherwise the PTY spawns wherever the Lanes app was launched from.
- ❌ Passing `--resume`, `--continue`, `--session-id`, or `--fork-session` to `lanes_start_session`. These are rejected — use `lanes_resume_session` to re-attach a stopped Claude session.
- ❌ Treating the ambiguity response from `lanes_stop_session` / `lanes_resume_session` / `lanes_delete_session` / `lanes_read_terminal` as an error. It's a normal MCP success result with `_meta.sessions` — pick a candidate and retry with `session` set.
- ❌ Confusing `cliSessionId` with `session`. `cliSessionId` is the CLI's own resume token (only consumed by `lanes_get_issue_history` and `lanes_get_session_stats`); `session` is the Lanes ref (UUID / slot / name) used by stop / resume / delete / read_terminal.
- ❌ **Re-calling `lanes_start_session` because you could not find the session you just started.** The launch is asynchronous and every call spawns *another* session. This is the single worst failure mode with these tools — it silently produces a pile of duplicate sessions on one issue. Read `lanes_get_session_status { issueId }` instead, and wait if it comes back empty.
- ❌ Treating `"No sessions exist for issue N yet"` as a cue to start a session. Inside the launch window it is the expected answer for a session that is starting normally.
- ❌ Using `lanes_get_issue` to check whether a session started. It does not return sessions — `lanes_get_session_status` does.
- ❌ Looking for only one of `status` / `runtimeStatus` on a session entry and concluding the session is broken when it's absent. Both keys are present and carry the same value.
- ❌ Creating a git worktree while running inside a Lanes session and never linking it. The issue keeps pointing at the main checkout, so `lanes_get_issue_changes` and the board's Changes tab both show an empty diff and your work looks like it never happened. Call `lanes_update_issue { id: $LANES_SESSION, branch: "<branch>" }` right after `git worktree add`, then re-read the issue to confirm `worktreeStrategy` is `"select"`.
