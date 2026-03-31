# Git Integration

> **Read before**: [File Browser & Editor](./file-browser.md)

Lanes provides a built-in Git viewer so you can see exactly what changed in a
worktree without switching to another tool. The Git integration lives inside the
issue detail panel and gives you both a high-level file list and a line-by-line
diff view.

## Opening the Git View

When you select an issue that has an associated worktree, the detail panel
includes a **Git Changes** subtab. Click it to see the current state of the
worktree compared to the configured base branch.

## Two Viewing Modes

The Git Changes subtab offers two modes, toggled by a control at the top:

### Changes Mode

Shows all **uncommitted and committed differences** between the worktree and its
base branch. This is the view you will use most often -- it answers the question
"what has this worktree changed so far?"

### History Mode

Shows the **commit log** for the worktree branch. Each commit is listed with its
message, author, and timestamp. Click a commit to see the specific files it
touched and their diffs.

## The File List Panel

On the left side of the Git view, you see a list of all modified files. Each
file entry includes:

- **Status badge** -- a letter indicating the type of change:
  - **M** -- Modified
  - **A** -- Added (new file)
  - **D** -- Deleted
  - **R** -- Renamed
- **Line counts** -- a summary showing how many lines were added (+) and
  removed (-), giving you a quick sense of the change size.

Files are grouped and sorted so you can scan the list and quickly identify
what was touched.

## Viewing Diffs

Click any file in the list to open its diff in the right-side panel. The diff
viewer uses the Monaco diff editor, showing a side-by-side comparison with:

- Added lines highlighted in green.
- Removed lines highlighted in red.
- Unchanged context lines surrounding each change.

This is the same diff experience you would get in VS Code, integrated directly
into Lanes.

## How Diffs Relate to Worktrees

The diff comparison always runs against the **configured base branch** for the
project. By default, Lanes auto-detects the base branch from Git (typically
`main` or `master`). You can override this in Settings if your project uses a
different convention.

This means:

- In **Changes mode**, you see everything the worktree branch has done relative
  to the base -- both committed and uncommitted work.
- In **History mode**, you see only the commits that exist on the worktree
  branch but not on the base branch.

## Switching Between Modes

Use the toggle at the top of the Git Changes subtab to switch between Changes
and History. Your selection is remembered while the issue stays open, so you can
flip back and forth as you work.

## Viewing a Specific Commit

In History mode, click any commit in the log to see:

1. The list of files that commit changed (with the same M/A/D/R badges).
2. The diff for each file, scoped to just that commit.

This is useful for reviewing an agent's work step by step -- you can see what
each commit did individually rather than looking at the cumulative diff.

## Tips

- Use Changes mode for a "big picture" review of everything the agent has done
  on this issue.
- Use History mode when you want to understand the progression of changes,
  commit by commit.
- The Git view updates in real time as the agent makes new commits, so you can
  watch progress without refreshing.
- If a diff looks unexpected, double-check that the base branch is set correctly
  in Settings. A wrong base branch can make the diff include unrelated changes.

---

**Read next**: [Process Manager](./process-manager.md)
