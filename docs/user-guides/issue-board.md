# The Issue Board

> **Read before**: [Quick Start](./quick-start.md)

The issue board is the heart of Lanes. It is the workspace where every issue — whether it is a planning note or a live AI session — has a visual home.


## Board Layout

The board occupies the center of the Lanes window:

- **Sidebar** (left) — Project navigation, worktree list, and quick access to settings. Toggle it open or closed for more board space.
- **Workflow columns** (center) — Vertical columns representing workflow stages. Issues appear as cards within these columns.
- **Detail panel** (bottom) — A resizable split view that shows the terminal on the left and issue details on the right. Click any issue card to open it here.


## Columns

Each column represents a stage in your workflow:

| Column | Purpose |
|--------|---------|
| **Backlog** | Issues waiting to be worked on. A holding area for future tasks. |
| **Planning** | Issues being scoped or analyzed. Agents started in plan mode land here. |
| **Implementation** | Active development. Agents doing the actual coding work live here. |
| **Review** | Work that needs human review before it can be marked done. |
| **Done** | Completed issues. Use the column action menu to clear finished items. |
| **Misc** | A catch-all for issues that do not fit the standard workflow. |

Columns are collapsible. Click the column header chevron to collapse or expand a column, which is useful when you want to focus on just a few stages.


## Creating Issues

Press **Cmd+N** (Ctrl+N on Linux) to open the new issue dialog. Every issue has:

- A **title** (required)
- An optional **description** for context
- **Instructions** that get sent to the AI agent as the initial prompt
- A **CLI** selection (Claude Code, Codex, Gemini, or a bare terminal)
- A **working directory** and optional **worktree strategy**

Issues start as drafts. Once you fill in the required fields and confirm, they appear on the board.


## Editing Issues

Click any issue card to open it in the detail panel. From there you can:

- **Edit the title** inline by clicking on it
- **Edit the description and instructions** in the Issue tab on the right side of the detail panel
- **View metadata and metrics** (tokens, cost, runtime) in the Meta tab

All edits save automatically as you type.


## Moving Issues Between Columns

There are two ways to move issues:

1. **Drag and drop** — Click and drag a card from one column to another. Drop it at the desired position within the target column.
2. **Right-click context menu** — Right-click a card and select **Move** to choose a target column from the submenu.

Lanes preserves your custom sort order within each column. When you drop a card between two others, it stays where you put it.


## Multi-Select

You can select multiple issues at once for bulk actions:

- **Shift+Click** — Select a range of issues between the last selected card and the one you click.
- **Cmd+Click** (Ctrl+Click on Linux) — Toggle individual cards in or out of the selection.
- **Cmd+A** (Ctrl+A) — Select all issues in the focused column. Press it again to select all issues on the entire board.
- **Escape** — Clear the current selection.

When multiple issues are selected, a **bulk actions toolbar** appears with options to:

- **Stop** — Stop all running sessions on the selected issues
- **Remove** — Delete the selected issues from the board
- **Complete** — Mark all selected issues as done


## Right-Click Context Menu

Right-click any issue card to access:

| Action | What it does |
|--------|-------------|
| **Move** | Move the issue to a specific column |
| **Complete** | Mark the issue as done and stop its session |
| **Stop Runtime** | Stop the running session without completing the issue |
| **Remove** | Delete the issue from the board |
| **Add Label** | Open the label picker to tag the issue |
| **Add Dependency** | Link this issue to others it depends on |
| **Move to Backlog** | Send the issue back to the Backlog column |


## Board Tabs

Above the columns, you will find tabs that filter the board view:

- **All** — Shows every issue across all projects and worktrees.
- **Per-project** — One tab per registered project. Shows only issues belonging to that project.
- **Per-worktree** — One tab per active worktree. Shows only issues running in that worktree.

Tabs are a quick way to focus on a specific area of work without setting up filters.


## Sort Options

Click the sort control in the board header to change the display order:

| Sort | Behavior |
|------|----------|
| **Default** | Manual sort order (drag-and-drop position) |
| **Newest** | Most recently created issues first |
| **Updated** | Most recently modified issues first |
| **Alphabetical** | Sorted A-Z by title |

The chosen sort order applies across all columns.

---

**Read next**: [Working with Sessions](./sessions.md)
