---
name: github-lanes-bridge
description: Use when bridging GitHub and Lanes — importing GitHub issues into Lanes for local Claude Code execution, batch-spawning sessions per ticket, decomposing one GitHub issue into multiple Lanes sub-issues with dependencies, or posting session results (PR links, comments, follow-up issues) back to GitHub. Triggers on phrases like "import from GitHub", "bring this GitHub issue into Lanes", "comment back on the PR/issue", "spin off a GitHub issue from this Lanes ticket", or any mention of both GitHub and Lanes in the same request. Uses the Lanes MCP's built-in lanes_github_* tools — no separate GitHub MCP needed (the user must have connected GitHub in Lanes settings).
---

# GitHub ↔ Lanes bridge

Use this skill to move work between GitHub (where issues and PRs live) and Lanes (the local agent execution board). Typical flows: pull an issue into Lanes, run a session on it, then push the outcome (a PR-linking comment, or a new follow-up issue) back to GitHub.

This skill assumes the **`lanes-sessions`** skill is also available — it covers the lanes_* issue/session tools in detail. This skill focuses on the **bridging** layer.

When a Lanes issue hosts more than one CLI session, follow the multi-session disambiguation rules in `lanes-sessions` (pass `session` on stop / resume / read_terminal). Bridge tools (`lanes_github_*`) never address sessions directly.

## When to use

- Work originates in GitHub and you want Claude Code to do the actual implementation locally.
- You want to fan out one GitHub issue into multiple Claude sessions on sub-tasks.
- You want PR links / progress updates to flow back to GitHub without leaving chat.

**Don't use** for ad-hoc one-off tasks that didn't start in GitHub; for those, just use Lanes directly.

## Prerequisites

- **Lanes MCP must be connected** — verify with `lanes_list_components`. If missing, run `/lanes:setup-mcp`.
- **GitHub must be connected inside Lanes** — verify with `lanes_github_list_repos`. If it returns a "not connected" error, ask the user to connect GitHub from **Lanes → Settings → GitHub** (OAuth flow). The token is read from `integrations.json` on every call. GitHub tokens don't expire, so no refresh logic to worry about.

There is no standalone GitHub MCP required. The same Lanes MCP that exposes `lanes_list_issues` also exposes `lanes_github_list_repos`, `lanes_github_get_issue`, etc. — they're additional tools on the same server.

## The GitHub tool surface

| Tool | Purpose |
|---|---|
| `lanes_github_list_repos` | List repos the user can access (owner, collaborator, or org member). Sorted by most-recently-pushed. Each is `{ id: "owner/name", name: "owner/name" }`. |
| `lanes_github_list_issues` | First 50 open issues in a repo, most-recently-updated first. PRs are filtered out automatically. |
| `lanes_github_search_issues` | Free-text search via the `/search/issues` endpoint. When the query is a pure number (`"#9"`, `"9"`), the exact-match issue is prepended. |
| `lanes_github_get_issue` | Single issue by **number** (passed as a string, e.g. `"42"`). |
| `lanes_github_create_issue` | Open a new issue. Labels are GitHub label **names** (not IDs). |
| `lanes_github_comment_on_issue` | Post a comment on an issue or PR. (GitHub uses the same endpoint for both.) |

Returned `ExternalIssue` shape:

```json
{
  "provider": "github",
  "externalId": "42",          // issue number as a string
  "externalKey": "#42",
  "externalUrl": "https://github.com/owner/name/issues/42",
  "title": "...",
  "description": "...",
  "state": "open",
  "updatedAt": 1715600000000
}
```

## Field mapping (GitHub → Lanes)

| GitHub | Lanes | Notes |
|---|---|---|
| `externalKey` (e.g. `#42`) | Title prefix + description marker | Used as the dedup key. |
| `title` | `title` | Pass through. |
| `body` (markdown) | `description` | Prepend a marker line: `GitHub: owner/name#42 — <externalUrl>` so round-tripping is trivial. |
| `state` (`open` / `closed`) | `step` | `open` → `backlog`/`todo` (your call); `closed` → `done` if explicitly importing closed issues. |
| upstream `externalId` (issue number) | `externalId` on `lanes_create_issue` | Pass through. |
| upstream `externalUrl` | `externalUrl` on `lanes_create_issue` | Pass through. |
| upstream `externalKey` | `externalKey` on `lanes_create_issue` | Pass through. |
| `labels` | `tags` | Different ID spaces. Look up Lanes label UUIDs via `lanes_list_labels` and map by name. Missing Lanes labels: ask user before creating. |

`lanes_create_issue` accepts `externalProvider`, `externalId`, `externalKey`, `externalUrl`, and `externalSyncedAt` — set all of them on import so the Lanes UI shows the badge and the refresh button works.

## Workflow patterns

### A. Single port (one GitHub issue → one Lanes session)

```
1. lanes_github_list_repos                                // (or user supplies "owner/name")
2. lanes_github_search_issues { repo: "owner/name", query: "#42" }
   or lanes_github_get_issue { repo, number: "42" }
3. lanes_list_issues { search: "owner/name#42" }          // dedup check
4. lanes_list_components                                  // pick componentId
5. lanes_create_issue {
     title: "owner/name#42 · <github title>",
     description: "GitHub: owner/name#42 — <externalUrl>\n\n<github body>",
     componentId,
     cwd,
     worktreeStrategy: "create",
     worktreeName: "gh/owner-name-42",
     externalProvider: "github",
     externalId: "42",
     externalKey: "#42",
     externalUrl: "<github url>",
     externalSyncedAt: <now epoch ms>
   }
6. lanes_start_session { issueId, planMode: true }
```

Confirm with the user before step 6 if the issue is non-trivial — let them review the issue first.

### B. Backlog import (filter GitHub → bulk Lanes issues)

```
1. lanes_github_list_issues { repo }   (capped at 50)
   or lanes_github_search_issues { repo, query: "label:p0" }
   Show the user the result count BEFORE importing.
2. Confirm scope with the user.
3. lanes_list_components, lanes_list_labels (cache locally)
4. For each GitHub issue:
     - dedup via lanes_list_issues { search: "<externalKey>" }
     - lanes_create_issue (with marker, mapped labels → tags, external* metadata)
5. (Optional) batch lanes_start_session in plan mode with
   flags: [{ flag: "dangerously-skip-permissions", value: "", kind: "flag" }]
```

For wider sweeps, prefer `lanes_github_search_issues` with explicit qualifiers (`label:foo`, `assignee:@me`, `milestone:"v2"`) over `list_issues` with no filter — GitHub's search syntax composes naturally inside the query string.

### C. Decomposition (one GitHub issue → many Lanes sub-issues)

```
1. lanes_github_get_issue { repo, number }                // big ticket
2. Propose a breakdown to the user (numbered list).
3. After user approves:
     a. lanes_create_issue for the parent (the umbrella, with GitHub marker).
     b. lanes_create_issue for each sub-task with
        dependencies: [parentLanesId].
4. lanes_start_session on the parent in plan mode (optional);
   start sub-tasks once their dependencies are unblocked.
```

If the team needs sub-issues visible upstream too, call `lanes_github_create_issue` for each and add their `#N` to each Lanes description.

### D. Round-trip (push results back)

When a session is done and there's something concrete to report:

```
1. lanes_get_issue_changes { id }                  // confirm there is a diff
2. (User pushes the branch / opens a PR.)
3. lanes_github_comment_on_issue {
     repo: "owner/name",
     number: "42",
     body: "Implemented locally via Lanes #<lanesId>. PR: <PR URL>\n\n<stat summary>"
   }
4. (Optional) lanes_github_create_issue for follow-up work surfaced during the
   session.
5. lanes_move_issue { id: lanesId, step: "done" }
```

PRs and issues share the comment endpoint on GitHub — you can use `lanes_github_comment_on_issue` to comment on a PR by passing its number.

## Idempotency

Always include `GitHub: owner/name#N` in the Lanes description (and ideally in the title prefix). Before importing, run `lanes_list_issues { search: "owner/name#N" }` and skip if a match is found. For backlog imports, do the dedup check per-issue, not just up-front, so partially-imported batches can resume cleanly.

## Common pitfalls

- ❌ **Passing the GitHub issue number as a JSON integer.** The tool expects a string (e.g. `"42"`, not `42`) so the same parameter shape works for arbitrary external-id forms.
- ❌ **Forgetting `external*` metadata on import.** Without them, the Lanes UI can't show the "open in GitHub" badge or refresh the description.
- ❌ **Using `lanes_github_list_issues` and expecting PRs.** The endpoint mixes issues and PRs, but the tool filters PRs out (matching the renderer). Use the GitHub UI or `gh pr list` for PRs — there's no `lanes_github_list_prs` tool today.
- ❌ **Passing GitHub label IDs to `lanes_github_create_issue`.** Labels are by **name** (string), not ID — GitHub's REST API expects names.
- ❌ **Skipping the dedup marker.** Without `GitHub: owner/name#N` in the description, a re-run double-imports the same ticket.
- ❌ **Auto-pushing branches or opening PRs without user consent.** Never do remote git operations in Pattern D without the user explicitly asking.
- ❌ **Posting back on every session start.** Round-trip happens at session *end* (work landed). Don't spam GitHub comments mid-run.
- ❌ **Assuming `lanes_github_*` are read-only.** `lanes_github_create_issue` and `lanes_github_comment_on_issue` write to upstream — always confirm with the user before invoking on production repos.
