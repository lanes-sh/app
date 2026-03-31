# Process Manager

> **Read before**: [Working with Sessions](./sessions.md)

The process manager is a dialog that shows every CLI process Lanes knows about.
It is your go-to tool for understanding what is running, cleaning up after
crashes, and reclaiming system resources.

## Opening the Process Manager

There are two ways to open it:

- Click the **process icon** in the status bar at the bottom of the window.
- Open it from the application menu.

The dialog appears as an overlay on top of your board.

## What It Shows

Each row in the process list displays:

- **Process name** -- the CLI that is running (e.g., "claude", "codex").
- **PID** -- the operating system process ID.
- **Associated issue** -- if the process is linked to an issue, its title
  appears here as a clickable link. You can jump straight to the issue from the
  process manager.
- **Classification** -- one of three categories described below.

## Process Classifications

Lanes organizes processes into three groups so you can quickly understand their
origin and status:

### Tracked

Processes that Lanes started and is actively managing. These are the normal
case -- each one corresponds to a running session attached to an issue. Lanes
monitors their lifecycle, captures their output, and tracks their metrics.

### Orphan

Processes that Lanes previously started, but whose session has ended or
disconnected. This can happen if Lanes exits unexpectedly, if a session crashes,
or if a process outlives its session for any reason. Orphans are still running
and consuming resources, but Lanes is no longer actively managing them.

### External

CLI processes discovered on your system that Lanes did not start. Lanes scans
for known CLI processes (like `claude` or `codex`) and lists them here for
visibility. This helps you spot agents running outside of Lanes that might
conflict with your work or consume resources.

## Killing a Single Process

Click the **kill button** next to any process to terminate it. Lanes sends a
graceful termination signal (SIGTERM) first. If the process does not exit
within a few seconds, it escalates to a forced kill (SIGKILL).

## Stop Sessions vs Kill All

At the top of the process manager, you have two bulk actions:

- **Stop Sessions** -- Sends a graceful stop signal to all **tracked** sessions.
  This is the polite option: it tells each agent session to wrap up and exit
  cleanly. Orphan and external processes are not affected.

- **Kill All** -- Terminates **every** process in the list, regardless of
  classification. Use this when you need a clean slate -- for example, after a
  crash left several orphaned processes behind.

## Refreshing the Process List

Click the **refresh button** to rescan for current processes. The list updates
to reflect any processes that have started or stopped since you opened the
dialog. This is useful if you killed a process externally or if a session just
finished.

## When to Use the Process Manager

You do not need the process manager during normal operation -- Lanes handles
session lifecycles automatically. It becomes useful in these situations:

- **Orphaned processes consuming resources** -- After a crash or unexpected
  quit, orphaned agent processes may still be running. Open the process manager
  to find and kill them.

- **Cleanup after crashes** -- If Lanes restarts after a crash, old processes
  may still be alive. The process manager shows them as orphans so you can
  clean up.

- **Seeing what is running** -- When you have multiple sessions going and want
  a single view of all active processes, their PIDs, resource usage, and which
  issues they belong to.

- **Spotting external agents** -- If something feels slow or a port is in use,
  check for external CLI processes that might be competing for resources.

## Tips

- Get in the habit of checking the process manager after a crash or force-quit.
  Orphaned AI agents can rack up API costs if left running.
- The status bar icon gives you a quick count of running processes without
  opening the full dialog.
- Use "Stop Sessions" for routine cleanup; reserve "Kill All" for situations
  where you want everything shut down.

---

**Read next**: [Settings & Configuration](./settings.md)
