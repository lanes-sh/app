<p align="center">
  <h1 align="center">Lanes</h1>
  <p align="center"><strong>One workspace. Many agents.</strong></p>
  <p align="center">Run many AI coding agents in parallel lanes: issues, worktrees, sessions, one board.</p>
  <p align="center">
    <a href="#install">Install</a> ·
    <a href="https://lanes.sh/docs/desktop/quick-start">Quickstart</a> ·
    <a href="https://lanes.sh/docs">Docs</a> ·
    <a href="https://discord.gg/B3f8QjqeBa">Discord</a>
  </p>
</p>

<br>

<p align="center">
  <img src="https://lanes.sh/assets/demo.gif?v=3" alt="Lanes: agent sessions, worktrees and diffs on one board" width="768" />
</p>

<br>

## You lost track three agents ago

Five agents, eight terminal tabs. One is waiting for input. One finished ten minutes ago and nobody noticed. Two are quietly doing the same work.

Agents got good at writing code. Nothing got good at keeping track of them.

## Install

```bash
brew install --cask lanes-sh/lanes/lanes && open -a Lanes
```

macOS Ventura or later. Universal binary, native on Apple Silicon and Intel. Lanes updates itself on launch.

The **[Quickstart](https://lanes.sh/docs/desktop/quick-start)** takes you from here to a first agent session. Join the **[Discord](https://discord.gg/B3f8QjqeBa)** for updates and questions.

## What it is

A native macOS app that puts every agent session on an issue board. Each card is a task. Each task holds live agent terminals, its own git worktree, and the diff those agents produced. You see what is running, what is blocked, what is waiting on you, and what shipped, in one window.

Review it without leaving that window: the diff, the editor, the database, the commit, the pull request.

## What you get

- **A board that holds the work.** Drag issues through Planning, Implementation, Review and Done. Multi-select, labels, filters, dependencies with cycle detection, and board tabs scoped per project so two repos never bleed into each other.
- **Real terminals, several per issue.** Every session is PTY-backed, and one card can hold the plan-mode session plus the two implementation runs it spawned. Live status, per-session token usage, and resume across restarts.
- **Claude Code and Codex, first class.** Lanes detects them, reads their state, resumes their sessions and lists their models. Anything else runs as a plain shell session.
- **A worktree per issue.** Generated branch names, uncommitted state in the status bar, setup and teardown scripts, and auto-cleanup when the issue completes.
- **A git client, not just a diff.** Commit, publish, switch branches, undo the last unpushed commit, and open a pull request from the issue panel.
- **Monaco and a database browser.** A real editor with a file tree, and read-only SQL against the SQLite files it finds in your project, so checking what an agent actually wrote is two clicks.
- **The model provider you choose.** [Gateway](https://lanes.sh/docs/desktop/gateway) points a session at Ollama, LM Studio, vLLM, OpenRouter or z.ai, applied as real process environment so the request goes straight from the CLI to your provider. [Local LLMs](https://lanes.sh/docs/desktop/local-llms) drives Ollama end to end. Both are research previews.
- **GitHub and Linear over OAuth.** Import a whole sprint onto the board, and push results back as comments.
- **A team, when you want one.** Remote workspaces sync issues, chat and presence in about two seconds, with assignees, presence avatars, and an honest banner when sync is degraded.

Full detail: **[Desktop docs](https://lanes.sh/docs/desktop)**.

## Your apps, memory and skills

[Lanes Link](https://lanes.sh/link) is one endpoint you run: Gmail, GitHub, Linear, Notion, Slack and over a hundred others, plus your memory, tasks and skills, behind one set of permissions.

- **Connect once, use everywhere.** Any MCP server, any REST API with an OpenAPI spec, any IMAP mailbox, any CalDAV server.
- **Permissions enforced at dispatch**, not asked of the model, with every call in an append-only audit log.
- **Profiles** keep work and personal credentials, memory and policy apart, and credentials never reach the agent.

Install it from inside Lanes, or run `bun install -g @lanes-sh/link`. Free and open source at [github.com/lanes-sh/link](https://github.com/lanes-sh/link).

## For agents

Lanes ships a built-in [MCP](https://modelcontextprotocol.io) server so Claude Code, Codex and other clients can read your board and drive sessions. Enable it under **Settings > Integrations > Lanes MCP**, then use **Connect Claude Code** or **Connect Codex**.

Once an agent can read session status and the diff that came out of it, you can stop prompting and start looping. [Building loops](https://lanes.sh/docs/desktop/loops) is the guide; [Loop engineering](https://lanes.sh/blog/loop-engineering-with-lanes) is the thinking behind it.

## Under the hood

Tauri 2, React 19, Rust and SQLite. Axum serves the MCP, `portable-pty` runs the terminals, xterm.js renders them, Monaco edits your files.

Local-first by default: a local workspace keeps every issue, session and setting in SQLite on your machine, and your code never goes anywhere. Remote workspaces are opt-in. Crash reporting is on by default and can be turned off under **Settings > User > General**.

## Docs

- **[Quickstart](https://lanes.sh/docs/desktop/quick-start)**, install to a running session.
- **[Desktop docs](https://lanes.sh/docs/desktop)**, including [the board](https://lanes.sh/docs/desktop/issue-board), [sessions](https://lanes.sh/docs/desktop/sessions), [worktrees](https://lanes.sh/docs/desktop/worktrees), [keyboard shortcuts](https://lanes.sh/docs/desktop/keyboard-shortcuts) and [settings](https://lanes.sh/docs/desktop/settings).
- **[All Lanes docs](https://lanes.sh/docs)**, where [/docs/mcp](https://lanes.sh/docs/mcp) compares the three MCP servers we ship.

## Part of Lanes

A family of products for building with agents. [Overview](https://lanes.sh/).

- **[Lanes Desktop](https://lanes.sh/desktop)** is this one: your agent sessions on a board, on your Mac.
- **[Lanes Link](https://lanes.sh/link)** is one MCP endpoint you run yourself, between your agents and your accounts, memory, skills and secrets. Open source.
- **[Lanes Forms](https://lanes.sh/forms)** gives you a live form endpoint from a single POST, with no signup.
- **[Lanes Compute](https://lanes.sh/compute)** is GPUs on demand, a single A100 through multi-node H100 clusters, billed per second. Limited early access.

## License

Proprietary. All rights reserved.
