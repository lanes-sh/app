# Quick Commands

> **Read before**: [Working with Sessions](./sessions.md)

Quick commands are pre-configured prompts or shell commands that you can execute
with a single click or keystroke. They let you automate repetitive actions --
committing code, running tests, linting -- without typing the same instructions
over and over.

## Built-in Commands

Lanes ships with a set of default quick commands so you can be productive
immediately:

| Command | What it does |
|---------|-------------|
| **Commit** | Asks Claude Code to commit the current changes |
| **Release** | Walks through versioning and release prep |
| **Review Changes** | Reviews uncommitted work against the base branch |
| **Add Tests** | Generates tests for recent changes |
| **Fix Lint** | Finds and fixes linting errors |
| **Refactor** | Suggests and applies refactoring improvements |
| **Test Worktree** | Runs the project test suite inside the worktree |
| **Complete & Merge** | Finishes the issue and merges the worktree branch |

You can modify or remove any of these, and add your own.

## Command Types

Every quick command has a **type** that determines how it runs:

- **Claude** -- Sends a prompt into the active Claude Code session. The prompt
  is injected into the terminal as if you typed it, so Claude Code picks it up
  and acts on it. Use this for anything you would normally ask an AI agent to do.

- **Terminal** -- Executes a raw shell command directly in the session terminal.
  Use this for deterministic scripts like `npm test`, `cargo build`, or
  `git push`.

## Command Categories

Commands are also grouped into two **categories** that control where they appear:

- **General** -- Available everywhere. These show up in the quick-command menu
  regardless of whether the issue has a worktree. Good for universal actions
  like "Commit" or "Fix Lint."

- **Worktree** -- Only shown when the selected issue has an active worktree.
  These are for operations that only make sense in a worktree context, like
  "Test Worktree" or "Complete & Merge."

## Running Quick Commands

There are two ways to fire a quick command:

### From the Sidebar

Click the lightning bolt icon in the sidebar to open the quick-command menu.
You will see a list of available commands filtered by the current context
(general commands always appear; worktree commands appear only when relevant).
Click one to run it.

When you run a quick command from the sidebar without an issue selected, Lanes
creates a new **misc issue** to host the session. This keeps your board
organized -- every command execution is tracked against an issue.

### With Keyboard Shortcuts

Press **Cmd+Alt+1** through **Cmd+Alt+9** to run the first through ninth quick
command by position. The numbering matches the order in your quick-command list,
so you can reorder them to put your most-used commands on the easiest shortcuts.

## Creating Custom Quick Commands

1. Open **Settings** (Cmd+, or the gear icon in the sidebar).
2. Switch to the **Quick Actions** tab.
3. Click the add button to create a new command.
4. Fill in the fields:
   - **Name** -- A short label that appears in the menu (e.g., "Deploy Staging").
   - **Prompt / Command** -- The text to send. For Claude-type commands this is
     the prompt; for Terminal-type commands this is the shell command.
   - **Type** -- Choose "Claude" or "Terminal."
   - **Category** -- Choose "General" or "Worktree."
5. Save your changes.

## Editing and Reordering

From the same Quick Actions tab in Settings, you can:

- **Edit** any command by clicking on it and changing its fields.
- **Reorder** commands by dragging them up or down in the list. The order here
  determines both the menu order and which keyboard shortcut (Cmd+Alt+1-9)
  maps to which command.
- **Delete** a command you no longer need.

Changes take effect immediately -- no restart required.

## Tips

- Combine Claude-type commands with detailed prompts for repeatable AI
  workflows. For example, a "Write Changelog" command could include specific
  formatting instructions.
- Use Terminal-type commands for build scripts, deployment commands, or
  anything that does not need AI involvement.
- Keep your most-used commands in positions 1-9 so the keyboard shortcuts
  stay convenient.

---

**Read next**: [File Browser & Editor](./file-browser.md)
