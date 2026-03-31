# File Browser & Editor

> **Read before**: [Quick Start](./quick-start.md)

Lanes includes a built-in file browser and code editor so you can view and edit
project files without leaving the app. This is especially useful when you want
to inspect what an AI agent changed, make a quick fix, or browse your codebase
alongside the board.

## Adding Projects to the Sidebar

Before you can browse files, Lanes needs to know which directories to show.

- Click the **+** button in the sidebar to add a project folder, or
- Open **Settings > General** and manage your working folders from there.

You can add multiple project directories. Each one appears as a top-level entry
in the sidebar file tree.

## The File Tree

The sidebar shows an expandable file tree for each project you have added.
Folders can be expanded and collapsed by clicking the arrow next to them. The
tree is **lazy-loaded** -- child entries are fetched only when you expand a
folder, so even large projects stay responsive.

### Default-Collapsed Directories

Some directories are collapsed by default because they are typically not useful
to browse manually:

- `node_modules`
- `.git`
- `target`
- `dist`

You can still expand them if you need to; they just start closed to keep the
tree manageable.

## Navigating Files and Folders

Click any folder to expand or collapse it. Click any file to open it in the
editor panel on the right side of the screen. The file tree highlights the
currently open file so you can see where you are.

## The Monaco Editor

When you open a file, it appears in a full-featured Monaco editor -- the same
editor engine that powers VS Code. You get:

- **Syntax highlighting** for all common languages and file types.
- **Line numbers** in the gutter for easy reference.
- **Theme matching** -- the editor follows your app theme (light or dark), so
  it blends naturally with the rest of Lanes.

## Tab Management

You can have multiple files open at the same time. Each open file gets its own
**tab** at the top of the editor area.

- **Click a tab** to switch to that file.
- **Click the X** on a tab to close it.
- Tabs appear in the order you opened them.

If you open many files, the tab bar scrolls so all your tabs remain accessible.

## Unsaved Changes

When you edit a file, its tab shows a **dot indicator** to let you know there
are unsaved changes. This gives you a visual cue before you navigate away or
close the tab.

## Saving Files

Press **Cmd+S** to save the current file. The dot indicator disappears once the
save completes, confirming your changes are written to disk.

## File Size Limits

For performance and safety, Lanes shows a **warning** when you try to open a
file larger than **5 MB**. Very large files (minified bundles, binary data,
large logs) can slow down the editor. You can still choose to open them, but
the warning lets you know what to expect.

## Tips

- Use the file browser alongside agent sessions to review changes as the AI
  makes them -- open the terminal in one panel and the edited file in another.
- The editor is great for quick tweaks. For heavy editing sessions, you might
  still prefer your full IDE, but for inspecting diffs and making small fixes
  Lanes has you covered.
- Drag a file from the file tree onto an open terminal to inject its path.
  Handy when you want to tell an agent to look at a specific file.

---

**Read next**: [Git Integration](./git-integration.md)
