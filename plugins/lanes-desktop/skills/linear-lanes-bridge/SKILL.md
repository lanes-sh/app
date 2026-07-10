---
name: linear-lanes-bridge
description: Use when bridging Linear and Lanes — importing Linear issues into Lanes for local Claude Code execution, batch-spawning sessions per Linear ticket, decomposing one Linear issue into multiple Lanes sub-issues with dependencies, or posting session results (PR links, comments, status updates) back to Linear. Triggers on phrases like "import from Linear", "bring this Linear ticket into Lanes", "run the sprint locally", "push results back to Linear", or any mention of both Linear and Lanes in the same request. Uses the Lanes MCP's built-in lanes_linear_* tools — no separate Linear MCP needed (the user must have connected Linear in Lanes settings).
---

# Linear ↔ Lanes bridge

Use this skill to move work between Linear (the team's source of truth) and Lanes (the local agent execution board). Typical flows: pull a ticket or a whole sprint into Lanes, run sessions on each, then push the outcome (comments, new tickets) back to Linear.

This skill assumes the **`lanes-sessions`** skill is also available — it covers the lanes_* issue/session tools in detail. This skill focuses on the **bridging** layer.

When a Lanes issue hosts more than one CLI session, follow the multi-session disambiguation rules in `lanes-sessions` (pass `session` on stop / resume / read_terminal). Bridge tools (`lanes_linear_*`) never address sessions directly.

## When to use

- Work originates in Linear and you want Claude Code to do the actual implementation locally.
- You want to fan out one Linear ticket into multiple Claude sessions on sub-tasks.
- You want PR links / progress updates to flow back to Linear without leaving chat.

**Don't use** for ad-hoc one-off tasks that didn't start in Linear; for those, just use Lanes directly.

## Prerequisites

- **Lanes MCP must be connected** — verify with `lanes_list_components`. If missing, run `/lanes:setup-mcp`.
- **Linear must be connected inside Lanes** — verify with `lanes_linear_list_teams`. If it returns a "not connected" error, ask the user to connect Linear from **Lanes → Settings → Linear** (OAuth flow). All `lanes_linear_*` tools read the user's token from `integrations.json` on every call, including a transparent refresh of expired access tokens.

There is no standalone Linear MCP required. The same Lanes MCP that exposes `lanes_list_issues` also exposes `lanes_linear_list_teams`, `lanes_linear_get_issue`, etc. — they're additional tools on the same server.

## The Linear tool surface

| Tool | Purpose |
|---|---|
| `lanes_linear_list_teams` | List teams the user can access. Each returns `{ id, name, meta }` where `meta` is the team key (e.g. `"ENG"`). |
| `lanes_linear_list_issues` | First 50 open issues in a team, most-recently-updated first. Completed/canceled are filtered out. |
| `lanes_linear_search_issues` | Free-text search across title/description, plus issue-number shortcut (`"LA-9"`, `"#9"`, `"9"`). |
| `lanes_linear_get_issue` | Single issue by **UUID** (not the `LA-9` identifier — the underlying UUID returned by list/search). |
| `lanes_linear_create_issue` | Open a new issue in a team. Returns the created issue, including its identifier and URL. |
| `lanes_linear_comment_on_issue` | Post a comment on a Linear issue by UUID. |

Returned `ExternalIssue` shape:

```json
{
  "provider": "linear",
  "externalId": "<uuid>",
  "externalKey": "ENG-123",
  "externalUrl": "https://linear.app/.../issue/ENG-123",
  "title": "...",
  "description": "...",
  "state": "In Progress",
  "updatedAt": 1715600000000
}
```

## Field mapping (Linear → Lanes)

| Linear | Lanes | Notes |
|---|---|---|
| `externalKey` (e.g. `ENG-123`) | Title prefix + description marker | Used as the dedup key. |
| `title` | `title` | Pass through. |
| `description` | `description` | Prepend a marker line: `Linear: ENG-123 — <externalUrl>` so round-tripping is trivial. |
| `state` (raw name) | `step` | Map to `backlog`/`todo`/`in-progress`/`done` based on best fit. `lanes_linear_*` returns the human-readable state name, not the type — map conservatively (when unsure, use `backlog`). |
| upstream `externalId` | `externalId` on `lanes_create_issue` | Pass the Linear UUID through so `ExternalLinkBadge` can refresh. |
| upstream `externalUrl` | `externalUrl` on `lanes_create_issue` | Pass through. |
| upstream `externalKey` | `externalKey` on `lanes_create_issue` | Pass through. |

`lanes_create_issue` accepts `externalProvider`, `externalId`, `externalKey`, `externalUrl`, and `externalSyncedAt` — set all of them on import so the Lanes UI shows the badge and the refresh button works.

## Workflow patterns

### A. Single port (one Linear issue → one Lanes session)

```
1. lanes_linear_list_teams                                // pick teamId
2. lanes_linear_search_issues { teamId, query: "ENG-123" }
3. lanes_linear_get_issue { id: "<uuid from step 2>" }    // full description
4. lanes_list_issues { search: "ENG-123" }                // dedup check
5. lanes_list_components                                  // pick componentId
6. lanes_create_issue {
     title: "ENG-123 · <linear title>",
     description: "Linear: ENG-123 — <externalUrl>\n\n<linear description>",
     componentId,
     cwd,
     worktreeStrategy: "create",
     worktreeName: "linear/eng-123",
     externalProvider: "linear",
     externalId: "<linear uuid>",
     externalKey: "ENG-123",
     externalUrl: "<linear url>",
     externalSyncedAt: <now epoch ms>
   }
7. lanes_start_session { issueId, planMode: true }
```

Confirm with the user before step 7 if the ticket is non-trivial — let them review the issue first.

### B. Sprint import (filter Linear → bulk Lanes issues)

```
1. lanes_linear_list_issues { teamId } or
   lanes_linear_search_issues { teamId, query: "<filter>" }
   Show the user the result count BEFORE importing.
2. Confirm scope with the user.
3. lanes_list_components, lanes_list_labels (cache locally)
4. For each Linear issue:
     - dedup via lanes_list_issues { search: "<externalKey>" }
     - lanes_create_issue (with marker, componentId, external* metadata)
5. (Optional) batch lanes_start_session in plan mode with
   flags: [{ flag: "dangerously-skip-permissions", value: "", kind: "flag" }]
```

`lanes_linear_list_issues` is capped at 50 — for larger sprints, narrow the query first or page via `lanes_linear_search_issues` with deliberate terms.

### C. Decomposition (one Linear issue → many Lanes sub-issues)

```
1. lanes_linear_get_issue { id }                          // big ticket
2. Propose a breakdown to the user (numbered list).
3. After user approves:
     a. lanes_create_issue for the parent (the umbrella, with Linear marker).
     b. lanes_create_issue for each sub-task with
        dependencies: [parentLanesId].
4. lanes_start_session on the parent in plan mode (optional);
   start sub-tasks once their dependencies are unblocked.
```

The parent stays linked to Linear via the marker. Sub-tasks are Lanes-only; if the team needs them visible in Linear, call `lanes_linear_create_issue` for each and add their identifiers to each Lanes description.

### D. Round-trip (push results back)

When a session is done and there's something concrete to report:

```
1. lanes_get_issue_changes { id }                  // confirm there is a diff
2. (User pushes the branch / opens a PR.)
3. lanes_linear_comment_on_issue {
     id: "<linear uuid>",
     body: "Implemented locally via Lanes #<lanesId>. PR: <URL>\n\n<stat summary>"
   }
4. (Optional) lanes_linear_create_issue for follow-up work surfaced during the
   session.
5. lanes_move_issue { id: lanesId, step: "done" }
```

Linear treats PR URLs in comments as first-class — they get a rich preview, so the comment is as good as a sidebar attachment for most flows.

## Idempotency

Always include `Linear: <EXTERNAL_KEY>` in the Lanes description (and ideally in the title prefix). Before importing, run `lanes_list_issues { search: "<EXTERNAL_KEY>" }` and skip if a match is found. For sprint imports, do the dedup check per-issue, not just up-front, so partially-imported sprints can resume cleanly.

## Common pitfalls

- ❌ **Calling `lanes_linear_get_issue` with the identifier** (`"LA-9"`). The tool takes the UUID. Get the UUID from `list_issues` / `search_issues` first.
- ❌ **Forgetting `external*` metadata on import.** Without them, the Lanes UI can't show the "open in Linear" badge or refresh the description.
- ❌ **Bulk-importing without filters.** Always narrow with `lanes_linear_search_issues` before fanning out.
- ❌ **Skipping the dedup marker.** Without `Linear: <KEY>` in the description, a re-run double-imports the same ticket.
- ❌ **Auto-creating PRs without user consent.** Never push to a remote or open a PR in Pattern D without the user explicitly asking.
- ❌ **Posting back on every session start.** Round-trip happens at session *end* (work landed). Don't spam Linear comments mid-run.
- ❌ **Assuming `lanes_linear_*` are read-only.** `lanes_linear_create_issue` and `lanes_linear_comment_on_issue` write to upstream — always confirm with the user before invoking on production data.
