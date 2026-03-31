# Worktree Management

> **Read before**: [Working with Sessions](./sessions.md)

Lanes uses git worktrees to give each issue its own isolated branch and working directory. This means multiple AI agents can work on different tasks in the same repository at the same time, without their changes colliding.


## What Are Git Worktrees?

A git worktree is a separate checkout of your repository that shares the same `.git` history but has its own working directory and branch. Think of it as a lightweight clone that lives alongside your main checkout. Changes in one worktree do not affect another.

Lanes leverages this feature so that when you have three agents working on three different issues in the same repo, each one operates in its own directory on its own branch. No merge conflicts mid-work, no accidental overwrites.


## Worktree Strategies

When creating an issue, you choose a worktree strategy that controls how Lanes sets up the working environment:

| Strategy | Behavior |
|----------|----------|
| **Default** | Uses your project-level setting (which defaults to "Create" for git repos). |
| **None** | The session runs directly in the project's current directory. No worktree is created. Use this for quick tasks that do not need isolation. |
| **Create** | Lanes automatically creates a new git worktree for this issue. The session runs inside that worktree with its own branch. |
| **Select** | Opens a picker so you can choose an existing worktree. Useful when you want to continue work in a worktree that was created earlier. |


## How Auto-Creation Works

When you choose the "Create" strategy (or leave it on "Default" for a git project), Lanes:

1. Creates a new directory under `.worktrees/` in your project root.
2. Names it using the pattern `{issueId}-{adjective}-{noun}` — for example, `407-fond-cave` or `312-bright-oak`.
3. Creates a new git branch based on the project's base branch (usually `main` or `develop`).
4. Points the session's working directory to this new worktree.

The `.worktrees/` directory keeps all worktrees organized in one place. Add it to your `.gitignore` if you do not want it tracked.


## The Worktree Status Popover

The status bar at the bottom of the Lanes window includes a worktree indicator. Click it to open the worktree status popover, which shows:

- All active worktrees grouped by project
- **Dirty indicators** next to worktrees that have uncommitted changes
- Quick navigation to jump to an issue associated with a worktree

This gives you an at-a-glance view of which worktrees need attention.


## Dirty-State Warnings

Lanes monitors worktrees for two kinds of uncommitted state:

- **Uncommitted changes** — Modified, added, or deleted files that have not been committed yet.
- **Unpushed commits** — Commits that exist on the worktree branch but have not been pushed to the remote or merged into the base branch.

When either condition is detected, the worktree is marked as "dirty" in the status popover. This matters most during cleanup — Lanes will warn you before removing a dirty worktree so you do not lose work.


## Finishing Work in a Worktree

When your AI agent is done working in a worktree, Lanes provides two built-in quick commands to wrap things up. You will find these in the terminal side panel as action buttons when the issue has a worktree attached.

**Test Worktree** — Runs your project's test suite inside the worktree. Use this to verify the agent's changes before merging. It is a good habit to run this as soon as the agent signals it is done, before moving on to the merge step.

**Complete & Merge** — Commits any remaining changes, merges the worktree branch into the base branch, and completes the issue. This is the final step that brings the agent's work back into your main codebase.

The recommended flow is:

1. The agent finishes its work and the session becomes idle or stops.
2. Click **Test Worktree** to validate the changes.
3. If tests pass, click **Complete & Merge** to merge and clean up.
4. If tests fail, restart the session or intervene manually to fix the issues, then repeat.

These commands are also available as keyboard shortcuts (Cmd+Alt and the command's position number) and from the sidebar quick command picker.


## Auto-Cleanup on Issue Completion

When you complete an issue that has a dedicated worktree (either via Complete & Merge or manually):

1. Lanes checks whether the worktree is clean (no uncommitted changes, no unpushed commits).
2. If the worktree is **clean**, it is automatically removed. The branch and directory are deleted.
3. If the worktree is **dirty**, Lanes shows a warning dialog explaining what uncommitted state exists. You can choose to remove it anyway or keep it for manual cleanup.

This keeps your project directory from filling up with stale worktrees over time.


## Per-Project Base Branch

Each project in Lanes has a base branch that new worktree branches are created from. Lanes auto-detects this (usually `main` or `master`), but you can override it:

1. Open **Settings**.
2. Find the project in the project list.
3. Change the **Base branch** field to your preferred branch (e.g., `develop`).

All new worktrees for that project will branch off the configured base branch.


## Worktree Board Tabs

The board tabs above the workflow columns include per-worktree tabs. Clicking a worktree tab filters the board to show only issues that are running in that specific worktree. This is helpful when you have many issues across multiple worktrees and want to focus on one area of work.

You can combine worktree tabs with label filters and other filtering options for even more focused views.


## Tips

- **Use worktrees for anything non-trivial.** If the agent will be editing files, give it an isolated worktree. Reserve "None" for read-only tasks like code review or research.
- **Check the status popover regularly.** It is the fastest way to spot worktrees that need commits pushed or changes committed before you forget.
- **Let auto-cleanup do its job.** If you commit and push your changes before completing an issue, the worktree will be removed automatically with no prompts.

---

**Read next**: [Labels & Filtering](./labels-and-filtering.md)
