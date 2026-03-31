# Working with Sessions

> **Read before**: [The Issue Board](./issue-board.md)

Sessions are the runtime engine of Lanes. Each session is a real pseudo-terminal (PTY) running an AI coding agent, attached to an issue on your board. You can run multiple sessions in parallel and watch them all from the board.


## What Is a Session?

A session is a live terminal process — backed by a PTY — that runs an AI CLI like Claude Code, Codex CLI, or Gemini CLI. It has its own working directory, scrollback buffer, and lifecycle state. Sessions are always linked to an issue, which gives you a single place to track the task, its output, and its results.

Sessions survive if you close the detail panel or switch focus to another issue. They keep running in the background, and you can reconnect at any time by clicking the issue card.


## Starting a Session

Select an issue on the board and look at the detail panel at the bottom of the screen. You will see two start buttons:

- **Plan** — Launches the AI agent in plan mode. The agent reads the issue instructions, analyzes the codebase, and proposes a plan without making changes. The issue moves to the Planning column. This is the safer choice when you want to review the approach first.
- **Implement** — Launches the agent in implementation mode. It begins working on the task immediately, making code changes as needed. The issue moves to the Implementation column.

Both modes send the issue's instructions as the initial prompt to the agent.


## Starting a Bare Terminal

Press **Cmd+T** to open a new bare terminal session. This creates a plain terminal (not attached to an AI agent) that you can use for running tests, checking logs, or any other shell task. Bare terminals are also attached to an issue for tracking purposes.


## The Terminal Panel

The terminal panel occupies the left side of the detail view at the bottom of the screen. It is a fully functional terminal emulator with:

- Real-time output streaming from the AI agent
- A scrollback buffer (approximately 50KB) so you can review earlier output
- Dynamic resizing — drag the divider between the terminal and the issue details panel to adjust the split

The right side of the detail view shows the Issue tab (notes and instructions) and the Meta tab (session metrics like tokens, cost, and runtime duration).


## Interacting with the Terminal

While a session is running, you can interact with it directly:

- **Type in the terminal** — Your keystrokes are sent directly to the running process. Use this to answer agent questions, approve plans, or provide additional context.
- **Drag and drop files** — Drag a file from your file manager (or the Lanes file browser) onto the terminal. Lanes injects the file path as text, which is useful for pointing the agent at specific files.


## Session Status Indicators

Each issue card on the board shows the current session state with a visual indicator:

| Status | Meaning |
|--------|---------|
| **Busy** | The agent is actively working — processing, writing code, or thinking. |
| **Awaiting input** | The agent has paused and is waiting for you to provide input or approval. |
| **Stopped** | The session was manually stopped. The issue remains on the board. |
| **Exited** | The agent process exited on its own (finished its work or encountered an end condition). |
| **Error** | The session encountered an error. Check the terminal output for details. |

These indicators update in real time, so you can monitor several agents at a glance from the board.


## Resuming Claude Sessions

When you stop and restart a Claude Code session, Lanes automatically passes the `--resume` flag. This tells Claude Code to pick up where it left off, using the conversation history from the previous run. You do not lose context when you need to pause and restart.


## Session History

The detail panel includes a History tab that shows a paginated transcript of the session. This is a JSONL-based viewer that records every message exchanged between you and the agent, along with tool calls and their results. Use it to review what the agent did after the fact, especially for long-running sessions.


## Stopping Sessions

There are several ways to stop a running session:

- **Stop button** — Click the stop button in the detail panel toolbar.
- **Right-click > Stop Runtime** — From the issue card context menu on the board.
- **Cmd+D** — Marks the issue as complete, which stops the session automatically.
- **Bulk stop** — Select multiple issues and use the bulk toolbar's Stop action.


## Completing an Issue

When you mark an issue as complete (via Cmd+D, the Complete context menu item, or dragging to Done):

1. The running session is stopped if one is active.
2. If the issue had a dedicated worktree, Lanes checks for uncommitted changes. If the worktree is clean, it is automatically removed. If it has uncommitted work, Lanes warns you before cleanup.
3. The issue moves to the Done column.

You can always reopen a completed issue by dragging it back to an earlier column and starting a new session.

---

**Read next**: [Worktree Management](./worktrees.md)
