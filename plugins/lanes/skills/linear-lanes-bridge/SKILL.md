---
name: linear-lanes-bridge
description: Use when bridging Linear and Lanes — importing Linear issues into Lanes for local Claude Code execution, batch-spawning sessions per Linear ticket, decomposing one Linear issue into multiple Lanes sub-issues with dependencies, or posting session results (PR links, comments, status updates) back to Linear. Triggers on phrases like "import from Linear", "bring this Linear ticket into Lanes", "run the sprint locally", "push results back to Linear", or any mention of both Linear and Lanes in the same request. Requires both the Linear MCP and the Lanes MCP connected.
---

# Linear ↔ Lanes bridge

Use this skill to move work between Linear (the team's source of truth) and Lanes (the local agent execution board). Typical flows: pull a ticket or a whole sprint into Lanes, run sessions on each, then push the outcome (PRs, comments, status) back to Linear.

This skill assumes the **`lanes-sessions`** skill is also available — it covers the lanes_* tools in detail. This skill focuses on the **bridging** layer.

## When to use

- Work originates in Linear and you want Claude Code to do the actual implementation locally.
- You want to fan out one Linear ticket into multiple Claude sessions on sub-tasks.
- You want PR links / progress updates to flow back to Linear without leaving chat.

**Don't use** for ad-hoc one-off tasks that didn't start in Linear; for those, just use Lanes directly.

## Prerequisites

Both MCPs must be connected in this Claude Code session:

- **Lanes MCP** — verify with `lanes_list_components`. If missing, run `/lanes:setup-mcp` (or follow the lanes-sessions skill's setup section).
- **Linear MCP** — verify by listing teams with the Linear MCP's `list_teams` tool. The exact tool name depends on which Linear MCP the user has installed (e.g. `mcp__claude_ai_Linear__list_teams` from Claude.ai, or a different prefix from the official Linear MCP). Use whichever is available.

If either is missing, stop and ask the user to install it before continuing.

## Field mapping (Linear → Lanes)

| Linear | Lanes | Notes |
|---|---|---|
| `identifier` (e.g. `ENG-123`) | Title prefix + description marker | Used as the dedup key. |
| `title` | `title` | Pass through. |
| `description` | `description` | Prepend a marker line: `Linear: ENG-123 — https://linear.app/<workspace>/issue/ENG-123` so round-tripping is trivial. |
| state `type` | `step` | `triage` / `backlog` → `backlog`; `unstarted` → `todo`; `started` → `in-progress`; `completed` / `canceled` → skip import (or `done` if explicitly requested). |
| `labels` | `tags` | Different UUID spaces. Call `lanes_list_labels`, match by name, keep a Linear→Lanes UUID dict for the session. Missing Lanes labels: ask user before creating. |
| `team` / `project` | `componentId` | Call `lanes_list_components` first; pick the closest match. |
| `parent` | `dependencies` | Lanes dependency is a Lanes issue ID. Resolve the parent first, store its Lanes ID, then create children with `dependencies: [parentLanesId]`. |
| `assignee` | — | Not stored on Lanes. Mention in the description if relevant. |
| `cycle`, `priority`, `estimate` | — | Not currently mapped. Drop or surface in description. |

**Status states are team-scoped in Linear.** Use the state's `type` field (always one of `triage`/`backlog`/`unstarted`/`started`/`completed`/`canceled`) for the mapping above; never match on the human-readable `name` (it varies per team).

## Workflow patterns

### A. Single port (one Linear issue → one Lanes session)

```
1. Linear list_issues   → find the ticket (or the user supplies the identifier)
2. Linear get_issue     → full description
3. lanes_list_issues    { search: "ENG-123" }       // dedup check
4. lanes_list_components → pick componentId
5. lanes_create_issue {
     title: "ENG-123 · <linear title>",
     description: "Linear: ENG-123 — <url>\n\n<linear description>",
     componentId,
     cwd,
     worktreeStrategy: "create",
     worktreeName: "linear/eng-123"
   }
6. lanes_start_session  { issueId, planMode: true }
```

Confirm with the user before step 6 if the ticket is non-trivial — let them review the issue first.

### B. Sprint import (filter Linear → bulk Lanes issues)

```
1. Linear list_issues with explicit filters:
     team / project / cycle / state.type ∈ {backlog, unstarted}
   Show the user the result count BEFORE importing.
2. Confirm scope with the user.
3. lanes_list_components, lanes_list_labels (cache)
4. For each Linear issue:
     - dedup via lanes_list_issues { search: "<identifier>" }
     - lanes_create_issue (with marker, mapped tags, componentId)
5. (Optional) batch lanes_start_session in plan mode with
   flags: [{ flag: "dangerously-skip-permissions", value: "", kind: "flag" }]
```

Always require an explicit filter at step 1. Linear projects accumulate hundreds of stale issues — never `list_issues` with no filter.

### C. Decomposition (one Linear issue → many Lanes sub-issues)

```
1. Linear get_issue                                       // big ticket
2. Propose a breakdown to the user (numbered list).
3. After user approves:
     a. lanes_create_issue for the parent (the umbrella).
     b. lanes_create_issue for each sub-task with
        dependencies: [parentLanesId].
4. lanes_start_session on the parent in plan mode (optional);
   start sub-tasks once their dependencies are unblocked.
```

The parent stays linked to Linear via the marker. Sub-tasks are Lanes-only; if the team needs them visible in Linear, create matching Linear sub-issues separately and add their identifiers to each Lanes description.

### D. Round-trip (push results back)

When a session is done and there's something concrete to report:

```
1. lanes_get_issue_changes { id }                  // confirm there is a diff
2. (User pushes the branch / opens a PR, or you do via gh.)
3. Linear create_attachment {
     issueId,
     url: "<PR URL>",
     title: "PR · <pr title>"
   }                                                // PR shows up in the Linear sidebar
4. Linear save_comment {
     issueId,
     body: "Implemented locally via Lanes #<lanesId>. Diff: <stat summary>."
   }
5. (Optional) Linear save_status_update on the team if a stand-up post
   makes sense, or move the Linear issue to In Review by saving a new
   state on the issue.
6. lanes_move_issue { id: lanesId, step: "done" }
```

Prefer `create_attachment` over comments for URLs — Linear surfaces attachments in the sidebar; comments get buried.

## Idempotency

Always include `Linear: <IDENTIFIER>` in the Lanes description (and ideally in the title prefix). Before importing, run `lanes_list_issues { search: "ENG-123" }` and skip if a match is found. For sprint imports, do the dedup check per-issue, not just up-front, so partially-imported sprints can resume cleanly.

## Common pitfalls

- ❌ **Bulk-importing without filters.** Always filter Linear list_issues by team + state.type + (project or cycle).
- ❌ **Matching Linear state by `name`.** Names vary per team ("Triage", "Backlog", "On deck"…). Use the `type` field.
- ❌ **Reusing label UUIDs across systems.** Linear label IDs are not Lanes label UUIDs. Always re-resolve via `lanes_list_labels`.
- ❌ **Overwriting Linear descriptions.** When pushing back, use `save_comment` or `create_attachment`. Don't `save_issue` to mutate the description — you'll clobber team context.
- ❌ **Skipping the dedup marker.** Without `Linear: ENG-XXX` in the Lanes description, a re-run double-imports the same ticket.
- ❌ **Auto-creating PRs without user consent.** Never push to a remote or open a PR in Pattern D without the user explicitly asking.
- ❌ **Posting back on every session start.** Round-trip happens at session *end* (work landed). Don't spam Linear comments mid-run.
