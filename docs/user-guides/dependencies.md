# Dependencies

> **Read before**: [The Issue Board](./issue-board.md)

Dependencies let you link issues that block each other. When issue B depends on issue A, Lanes knows that B cannot proceed until A is done. This keeps your agents from starting work that relies on unfinished prerequisites.


## What Dependencies Are

A dependency is a directional link between two issues. It says: "this issue cannot be worked on until that issue is complete." For example:

- "Add API endpoint" must be done before "Build frontend form that calls the API"
- "Set up database migrations" must be done before "Implement user authentication"

Dependencies are one-to-many. An issue can depend on multiple other issues, and multiple issues can depend on the same prerequisite.


## Adding Dependencies

There are two ways to add a dependency:

### From the Context Menu

1. Right-click an issue card on the board.
2. Select **Add Dependency** from the context menu.
3. The dependency picker dialog opens, showing all issues in your board.
4. Check one or more issues that this issue depends on.
5. Confirm your selection.

### From the Issue Detail Panel

1. Click an issue to open it in the detail panel.
2. In the issue details on the right side, find the dependencies section.
3. Click to open the dependency picker.
4. Select the prerequisite issues and confirm.


## The Dependency Picker

The dependency picker is a multi-select dialog that lists all issues in your workspace. You can:

- Scroll through the full list of issues
- Check multiple issues to add several dependencies at once
- Uncheck issues to remove existing dependencies

The picker shows issue titles and their current workflow status so you can make informed choices.


## Removing Dependencies

To remove a dependency, open the dependency picker for the issue (via context menu or detail panel) and uncheck the issue you want to unlink. The dependency is removed immediately.


## Cycle Detection

Lanes automatically prevents circular dependency chains. If issue A depends on B, and B depends on C, you cannot make C depend on A. The cycle detection runs whenever you try to add a dependency, and it will block the addition if it would create a loop.

This guarantees that your dependency graph is always a directed acyclic graph (DAG), which means there is always a valid order in which issues can be completed.


## Visual Dependency Badges

Issue cards on the board display a small dependency badge when they have dependencies. The badge shows the number of issues that this card depends on. For example, a badge showing "2" means the issue is waiting on two other issues.

This gives you a quick visual signal when scanning the board — cards with dependency badges may be blocked and waiting for prerequisite work to finish.


## How Blocking Works

An issue with unfinished dependencies is considered **blocked**. Here is how blocking behaves:

- A dependent issue stays blocked until **all** of its prerequisite issues reach the Done column.
- Blocked issues show a distinct visual state on the board so you can tell at a glance which issues are waiting.
- You can still manually drag a blocked issue between columns if you want to override the blocking behavior.
- Once all prerequisites are complete, the issue is automatically unblocked.

For example, if issue C depends on both A and B, then C remains blocked until both A and B are in the Done column.


## Dependencies and Automation

Dependencies integrate with Lanes' automation features:

- **Queue manager** — When the queue manager auto-fills session slots from the backlog, it skips blocked issues. Only unblocked issues are eligible for automatic pickup.
- **Auto-pickup** — Blocked issues are not picked up for automatic session creation, even if they are marked as ready. They wait until their dependencies are resolved.

This means you can set up a chain of dependent issues in the backlog, and Lanes will work through them in the correct order automatically. As each issue completes and unblocks the next, the queue manager picks up the newly unblocked work.


## Tips

- **Keep dependency chains short.** Long chains (A > B > C > D > E) create bottlenecks. If possible, structure work so that issues can run in parallel.
- **Use dependencies for real blockers only.** If two tasks are merely related but can be done in any order, do not add a dependency. Labels or descriptions are better for tracking loose relationships.
- **Check the board for blocked issues.** After completing a major prerequisite, glance at the board to see what got unblocked. If automation is enabled, those issues may already be starting up.

---

**Read next**: [Quick Commands](./quick-commands.md)
