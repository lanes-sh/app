# Settings & Configuration

> **Read before**: [Quick Start](./quick-start.md)

Lanes is configurable out of the box. The Settings dialog gives you control over
appearance, project directories, workflow behavior, labels, quick commands, CLI
flags, and permissions -- all in one place.

## Opening Settings

- Press **Cmd+,** (the standard macOS shortcut), or
- Click the **gear icon** in the sidebar.

Settings are organized into tabs across the top of the dialog.

## General Tab

The General tab covers the basics:

### Theme

Choose between **Light**, **Dark**, or **System**. The System option follows
your operating system preference and switches automatically.

### Working Folder

Add or remove **project directories** that appear in the sidebar file browser.
Click the add button to pick a folder from your file system. Each directory you
add becomes a browsable project in the sidebar.

### Default Worktree Strategy

Controls what happens when you start a new session from an issue:

- **None** -- No worktree is created. The session runs in the project root.
- **Create** -- Lanes automatically creates a new Git worktree for the issue.
- **Select** -- Lanes prompts you to choose an existing worktree or create one.

### Default Start Mode

Sets the initial mode for new agent sessions:

- **Plan** -- The agent starts in planning mode, where it outlines an approach
  before making changes.
- **Implement** -- The agent starts in implementation mode and begins working
  immediately.

### Auto-Update

Toggle automatic updates on or off. When enabled, Lanes checks for new versions
and installs them in the background.

## Workflows Tab

Displays the current workflow steps that your issues move through:

**Planning** -- **Implementation** -- **Review** -- **Done**

This tab shows the configured step sequence. Workflow steps define the columns
on your board and the lifecycle stages for every issue.

## Labels Tab

Manage the labels you use to categorize issues:

- **Create** a new label by clicking the add button, giving it a name, and
  picking a color.
- **Edit** an existing label by clicking on it to change the name or color.
- **Delete** a label you no longer need.

Lanes offers **13 colors** to choose from, giving you enough variety to create a
meaningful color-coding system.

You can also manage labels from the board itself -- right-click any issue and
use the **Add Label** option in the context menu. But the Settings tab is the
place for bulk management: renaming, recoloring, and cleaning up old labels.

## Quick Actions Tab

This is where you create, edit, and organize your quick commands:

- **Create** a new command by clicking the add button.
- **Set its properties**: name, prompt or shell command, type (Claude or
  Terminal), and category (General or Worktree).
- **Edit** an existing command by clicking on it.
- **Reorder** commands by dragging them up or down. The order here determines
  the menu order and which keyboard shortcut (Cmd+Alt+1-9) maps to each
  command.
- **Delete** commands you no longer use.

For a deeper look at how quick commands work, see
[Quick Commands](./quick-commands.md).

## CLI Flags Tab

Add **persistent flags** that are passed to every CLI session Lanes starts.
Each flag is a key-value pair in the format `flag=value`.

For example, if you want every Claude Code session to run with permissions
bypassed, you could add:

```
--dangerously-skip-permissions
```

Flags set here apply globally to all sessions. Use this for settings you always
want active rather than typing them each time.

## Permissions Tab

This tab helps you verify that Lanes has the system access it needs:

### macOS Full Disk Access

Lanes needs Full Disk Access to manage terminals, create worktrees, and read
project files across your system. This tab shows whether Full Disk Access is
currently granted and walks you through enabling it in System Settings if not.

### Folder Access Verification

Confirms that Lanes can read and write to your configured project directories.
If a folder is inaccessible (moved, deleted, or permission-restricted), this
tab flags the issue.

## Per-Project Base Branch

Each project has a **base branch** used for Git diff comparisons. Lanes
auto-detects this from your repository (typically `main` or `master`), but you
can override it manually if your project uses a different convention (e.g.,
`develop` or `trunk`).

The base branch setting affects:

- What the Git Changes view compares against.
- How worktree branches are diffed.

You can set this from the project settings or it will be picked up automatically
the first time you add a project directory.

## Tips

- Set your theme to System so Lanes follows your OS dark/light preference
  automatically.
- Add all your active project directories upfront -- switching between projects
  is faster when they are already in the sidebar.
- Use the CLI Flags tab sparingly. Flags apply to every session, so only add
  ones you truly want everywhere.
- Review the Permissions tab if sessions fail to start or files are unreadable.
  Missing Full Disk Access is a common culprit on macOS.

---

**Read next**: [Keyboard Shortcuts](./keyboard-shortcuts.md)
