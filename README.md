<!-- ![Lanes Logo](logo.png) -->

<p align="center">
  <h1 align="center">Lanes</h1>
  <p align="center"><strong>Mission control for AI coding agents.</strong></p>
  <p align="center">Track, orchestrate, and ship across multiple AI CLI sessions from one place.</p>
  <p align="center"><a href="#install">Quick start</a></p>
</p>

<br>

<p align="center">
  <img src="assets/demo.png" alt="Lanes Dashboard" width="800" />
</p>

<br>

## You lost track three agents ago.

You have five AI coding agents running across eight terminal tabs. One is waiting for input. One finished ten minutes ago and you didn't notice. Two are doing overlapping work. The only thing keeping it all together is your short-term memory, and it just failed.

AI agents got powerful. The workflow around them didn't.

## Lanes fixes that.

Lanes is a native desktop app that puts every AI coding session on an issue board. Each card is a task. Each task can have a live agent terminal attached. You see what's running, what's blocked, what's waiting, and what shipped, in one window, at a glance.

No context switching. No lost terminals. No wondering what that tab was doing. Just drag work through your pipeline while your agents execute.

It's the layer that was missing between you and your fleet of AI agents.

## Install

```bash
brew install --cask lanes-sh/lanes/lanes && open -a Lanes
```

Requires macOS Ventura or later. Native on Apple Silicon and Intel.

**Join our [Discord community](https://discord.gg/B3f8QjqeBa)** for updates, to share feedback, ask questions, and connect with other users. We're iterating fast and Discord is the best way to stay in the loop.

## Features

### Issue Board
Drag issues through workflow columns, Planning, Implementation, Review, Done, plus Backlog and Misc. Multi-select with Shift+Click and Cmd+Click for bulk operations. Right-click context menus, sorting options, and collapsible columns. Board tabs scoped per project directory and worktree.

### Live Embedded Terminals
Every issue runs an AI session in a real PTY-backed terminal. Start sessions in plan mode or implement mode. Resume Claude sessions across restarts. Drag files onto the terminal to inject paths. Real-time status detection: busy, awaiting input, stopped, exited, error.

### Worktree Management
Auto-create git worktrees per issue with generated branch names, or select existing worktrees. Status overview in the status bar showing uncommitted and unmerged state. Auto-cleanup on issue completion. Per-project base branch detection with manual override.

### Labels & Filtering
Create, rename, and assign labels with 13 color options. Filter the board by label, working directory, or workflow step. Clear all filters in one action.

### Dependencies
Link issues as dependencies via a multi-select picker. Cycle detection prevents circular chains. Dependent issues stay blocked until all prerequisites reach Done.

### Quick Commands
Preset and custom commands with keyboard shortcuts Cmd+Alt+1–9. Two types: `claude` (injected into CLI session) and `terminal` (run as shell command). Built-in defaults shipped with new installs, fully customizable in Settings.

### File Browser & Editor
Sidebar file tree for the selected issue's working directory. Monaco editor with tabbed editing, dirty file tracking, syntax highlighting, and save on Cmd+S.

### Git Integration
Diff view with two modes: Changes (uncommitted working tree diff) and History (committed diffs). Monaco-powered inline diff viewer. Automatic branch detection from the issue's worktree.

### Process Manager
Discover running CLI processes across the system. Three-way classification: Tracked (managed by Lanes), Orphan (has issue ID but no active session), External (unrelated). Kill individual processes or stop all sessions at once.

### Works with

**Claude Code**, Anthropic's CLI for Claude.

Support for Codex CLI, Gemini CLI, and other terminal-based AI tools is on the roadmap.

## Auto-updates

Lanes checks for updates on launch and updates itself. You can also check manually in **Settings > About > Check Now**.

## Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| Cmd+N | New backlog issue |
| Cmd+T | New Misc task |
| Cmd+D | Complete selected issue(s) |
| Cmd+, | Open Settings |
| Cmd+S | Save file in editor |
| Cmd+A | Select all in column, then all issues |
| Cmd+Alt+1-9 | Run quick command by position |
| Shift+Click | Range select issues |
| Cmd/Ctrl+Click | Toggle individual issue selection |
| Escape | Clear selection or close dialog |

## Documentation

[User Guide](docs/user-guides/README.md), everything you need to know to get the most out of Lanes

## Built for developers who ship

Tauri 2. React 19. SQLite. Local-first architecture. Your code and data never leave your machine. Fast startup, low memory footprint, native performance.

## License

Proprietary. All rights reserved.
