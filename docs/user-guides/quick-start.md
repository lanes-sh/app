# Quick Start

Get your first AI session running in about two minutes once Lanes is installed.


## Install Lanes

Install Lanes via Homebrew:

```bash
brew install --cask lanes-sh/lanes/lanes && open -a Lanes
```

Requires macOS Ventura or later. Native on Apple Silicon and Intel.

On first launch, Lanes will detect which AI CLIs you have installed (Claude Code, Codex CLI, Gemini CLI) and display their versions in Settings.

**Join our [Discord community](https://discord.gg/B3f8QjqeBa)** for updates, to share feedback, ask questions, and connect with other users. We're iterating fast and Discord is the best way to stay in the loop.


## Set Your Working Directory

When Lanes opens for the first time, it will ask you to choose a working directory. This is the root folder where your projects live. You can change it later in Settings, and you can also set a per-issue working directory when creating individual issues.

Pick the folder where you keep your code repositories. Lanes will use this as the default location for new sessions.


## Tour the Interface

The main window has three areas:

- **Sidebar** (left) — Lists your projects, worktrees, and provides quick navigation. Collapse it with the toggle button at the top for more board space.
- **Board** (center) — A task board with columns representing workflow stages. This is where your issues live.
- **Detail panel** (bottom) — Opens when you select an issue. Shows the terminal on the left and issue details (notes, instructions, metadata) on the right. Drag the divider to resize either side.

The status bar at the very bottom shows connection state, active session count, total tokens used, and estimated cost.


## Create Your First Issue

Press **Cmd+N** (or Ctrl+N on Linux) to open the new issue dialog. Fill in:

- **Title** — A short description of the task, e.g., "Add dark mode toggle to settings page"
- **Description** — Optional context for the task
- **Instructions** — The prompt that will be sent to the AI agent when a session starts
- **CLI** — Which AI agent to use (Claude Code, Codex CLI, or Gemini CLI)
- **Working directory** — Defaults to your global setting; override it per issue if needed
- **Worktree strategy** — Choose "Create" to give this issue its own isolated git branch, or "None" to work in the current directory

Click **Create** to add the issue to your board. It will appear in the Backlog or Planning column.


## Start a Session

With your issue selected on the board, look at the detail panel at the bottom. You will see two buttons for starting a session:

- **Plan** — Starts the AI agent in plan mode. The agent will analyze the task and propose a plan before making any changes. Good for understanding the scope of work first.
- **Implement** — Starts the agent in implementation mode. It will begin working on the task immediately.

Click one of these buttons. Lanes will spawn a terminal session, attach it to your issue, and the AI agent will start running. You can watch its output in real time in the terminal panel.


## Interact with the Running Session

While a session is running, you can:

- **Type directly** in the terminal panel to send input to the agent
- **Drag files** from your file manager onto the terminal to inject their paths
- **Watch the status indicator** on the issue card — it shows whether the agent is busy, awaiting your input, or has stopped

The issue card on the board updates in real time to reflect the session state.


## Stop a Session

To stop a running session:

- Click the **Stop** button in the detail panel toolbar
- Or press **Cmd+D** to mark the issue as complete, which stops the session and (if a worktree was created) offers to clean up the worktree

You can also right-click an issue card on the board and select **Stop Runtime** from the context menu.


## What's Next

You now know the basics: create issues, start AI sessions, and watch them work on your board. From here, explore how the board organizes your work across multiple concurrent agents.

---

**Read next**: [The Issue Board](./issue-board.md)
