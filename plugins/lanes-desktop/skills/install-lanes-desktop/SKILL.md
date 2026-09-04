---
name: install-lanes-desktop
description: Use when installing, updating, or repairing the Lanes desktop app on a Mac, or when someone wants the Lanes issue board running for the first time. Triggers on "install Lanes", "set up Lanes", "get Lanes running", "download Lanes", "update Lanes", "Lanes won't open", or any request to connect an agent to a Lanes app that is not installed yet. Covers the Homebrew cask, first launch, adding a project, and handing off to /lanes:setup-mcp for the local MCP server.
---

# Install Lanes Desktop

Lanes is a native macOS app that puts every AI coding session on an issue board: each card is a task, and each task holds live agent terminals, its own git worktree, and the diff those agents produced.

Your job in this skill is to get from nothing to a running app with a project in it, and then to an agent that can drive it. Three steps, one of which is the user's rather than yours.

**Homebrew is the only supported install, and it is also the update path.** There is no `.dmg` and no direct download. If you cannot find a cask, that is the answer, not a reason to look for a binary.

## Preflight

Run all four before installing. Each failure has a different answer.

| Check | Command | A failure means |
|---|---|---|
| Platform | `uname -s` | Anything but `Darwin`: stop. Lanes Desktop is macOS only today. Point at [the docs](https://lanes.sh/docs/desktop) and offer Lanes Link instead, which runs anywhere. |
| macOS version | `sw_vers -productVersion` | Below 13 (Ventura): stop. There is no build for it. |
| Already installed | `test -d /Applications/Lanes.app && echo present` | Present: skip step 1. Go to "Update and repair" if they asked to update, or straight to step 2. |
| Homebrew | `command -v brew` | Missing: install it from https://brew.sh, then come back. Do not hand-roll an install. |

## 1. Install it

One command. The `open -a` is chained on so a first install ends with the app running rather than with a cask sitting in `/Applications`.

```
brew install --cask lanes-sh/lanes/lanes && open -a Lanes
```

`lanes-sh/lanes` is a tap, so the first run also taps it. Expect a download of a few hundred megabytes; do not treat a slow first run as a hang.

Verify:

```
test -d /Applications/Lanes.app && pgrep -x Lanes >/dev/null && echo running
```

If the app installed but is not running, `open -a Lanes` again. macOS may hold the first launch behind a Gatekeeper prompt the user has to answer on screen.

## 2. Point it at your code

**This step is theirs, not yours.** The welcome screen needs a folder picker, and you cannot click it.

Tell them: the welcome screen offers four ways in, and the one to pick is **Add a project**. Choose the folder your repos live in. That becomes the default location for new sessions, and it can be changed later under **Settings > General**, or per issue.

Then wait. Do not go on to step 3 until they say the app is open with a project in it. The MCP server has nothing to serve before that.

## 3. Connect your agent

Run the slash command that ships in this same plugin:

```
/lanes:setup-mcp
```

It registers the SSE server at `http://localhost:5353/sse` with Claude Code and with Cursor, skips whichever is not installed, and is safe to run twice. **Use it rather than writing the registration yourself.** It knows the three names the server has shipped under, and it never renames an existing entry, because a server's config name is the prefix on its tool names and a rename resets the user's tool approvals.

Only if that command is unavailable, because the skills were installed without the plugin:

```
claude mcp add --transport sse lanes-desktop http://localhost:5353/sse --scope user
```

Then have them open a fresh terminal so Claude Code loads it.

Verify the endpoint:

```
curl -sS -m 2 -o /dev/null -w "%{http_code}\n" http://localhost:5353/sse
```

`200` means Lanes is up and the MCP is reachable. Connection refused means either the app is not running, or the server is switched off under **Settings > Integrations > Lanes MCP**. Ask which before assuming a broken install.

## Update and repair

| Situation | What to run |
|---|---|
| Update | `brew upgrade --cask lanes-sh/lanes/lanes`. The app also updates itself on launch, so this is for when someone wants it now. |
| Reinstall over a broken install | `brew reinstall --cask lanes-sh/lanes/lanes` |
| Uninstall | `brew uninstall --cask lanes-sh/lanes/lanes`. Board data lives outside the app and survives. |
| Confirm what is installed | `brew list --cask --versions lanes` |
| App will not open | Check `sw_vers -productVersion` against Ventura first, then reinstall. Do not delete `/Applications/Lanes.app` by hand; Homebrew loses track of it. |

## Critical gotchas

- **A `200` from the probe says the app is up, not that a workspace exists.** Someone who skipped step 2 has a reachable MCP over an empty board, and the first `lanes_list_issues` will look broken to them.
- **The MCP server is a toggle, and it is off until it is turned on.** **Settings > Integrations > Lanes MCP**. Nothing in step 1 or 2 turns it on.
- **The port is configurable.** If `5353` refuses while the app is plainly running, ask them what the Integrations panel says before debugging anything else.
- **A `lanes` entry in an MCP config that does not point at `localhost:5353` is a different server.** That is the remote Lanes Forms one. Leave it alone.
- **`brew install` without the chained `open -a Lanes`** leaves a cask in `/Applications` and nothing running, and every later check then fails for a reason that has nothing to do with the install.

## Anti-patterns

- Do not go looking for a `.dmg`, a GitHub release asset, or a direct download. Homebrew is the install path and the update path.
- Do not register the MCP by hand when `/lanes:setup-mcp` is available, and never rename an entry that is already there.
- Do not run `/lanes:setup-mcp` before the app is open. The probe will fail and you will debug an installation that is fine.
- Do not create issues or start sessions as a victory lap. That is the `lanes-sessions` skill, and it is the user's call whether to go there next.

## Then what

- `lanes-sessions`, in this plugin, is the board and the sessions: creating issues, launching Claude Code against them, reading terminals.
- `github-lanes-bridge` and `linear-lanes-bridge` pull tickets in from either tracker.
- [Quickstart](https://lanes.sh/docs/desktop/quick-start) is the same path written for a person.
