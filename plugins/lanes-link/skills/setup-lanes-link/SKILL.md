---
name: setup-lanes-link
description: Use when installing or standing up Lanes Link, the self-hostable MCP endpoint that gives an agent someone's own accounts, memory, tasks, assets, skills, identity and vault. Triggers on "set up Lanes Link", "install Lanes Link", "connect my Gmail/Calendar/Notion to Claude", "one MCP for all my accounts", "lanes link start", "lanes link connect", a 401 from a Lanes Link endpoint, or an empty tool list after registering one. Covers the CLI install, sign-in, workspace choice, profiles and members, connecting accounts, and registering the endpoint with every agent on the machine.
---

# Set up Lanes Link

Lanes Link is one MCP endpoint the user runs themselves. Behind it sit the accounts they connect (Gmail, GitHub, Linear, Notion, Slack, and over a hundred more) plus material that is theirs rather than an account: memory, tasks, assets, skills, an identity record, and a vault for passwords and API keys. Nothing routes through Lanes servers.

Your job is the setup path, end to end, in the order below. **The order is load-bearing** and two of the steps fail silently when they are done out of turn. Read "Critical gotchas" before you start, not after something looks broken.

Once it is running, the endpoint installs its own usage skill at `~/.claude/skills/lanes-link/SKILL.md`. That document, not this one, is how to *use* what you set up here.

## Preflight

| Check | Command | A failure means |
|---|---|---|
| Bun | `bun --version` | Below 1.3.11, or missing: install from https://bun.com first. There is no build step and no other runtime. |
| Already installed | `command -v lanes && lanes --version` | Already there: skip step 1, and consider `bun update -g @lanes-sh/link`. |
| Already running | `lanes link status --json` | Answers: it is set up. Find out what they actually want changed before running anything. |

## 1. Install the CLI, and check your PATH

```
bun install -g @lanes-sh/link
lanes --version
```

**Run that second line.** Several commands read the token with `$(lanes link token show --raw)`, and with `lanes` off the PATH that substitutes to an empty string. The only symptom is a 401 that looks like a bad token, and it will cost an hour if you skip this.

Then sign in. A profile declares who may consume it, so the endpoint has to know who is asking. The network is needed to sign in and to refresh, not per call.

```
lanes auth login
```

## 2. Choose a workspace, and start the endpoint

A workspace holds the connections and the profiles, and decides which stores open them. Locally that is a directory and an encrypted file; deployed, a bucket and Secret Manager.

**Ask before choosing. This is the one decision that is expensive to change:** accounts authorised against `local` have to be migrated if they deploy later.

| Mode | Command | Who it is for |
|---|---|---|
| Local | `lanes link start --workspace local` | Agents on this machine: Claude Code, Codex, Cursor. Leave it running. |
| Self-hosted | `lanes link deploy --workspace cloud` | Anything that cannot reach this machine: claude.ai, ChatGPT, a phone. Goes to Google Cloud Run. Needs `gcloud` and a billing account; it creates the project itself. |

Every command from here down carries `--workspace <name>`, and this step is what fixes that name. A deployed workspace has its own credential store, so its token is a different string from the local one.

Optional, and only if they want the Lanes dashboard to read this endpoint:

```
lanes link pair --workspace local
```

It installs a local certificate first, and asks before it does. `pair` starts nothing: someone who ran only `pair` has a dashboard reporting the endpoint unreachable while it is answering perfectly well.

## 3. Create a profile, and put them on it

```
lanes link profile add personal --workspace local
lanes link profile members add --me --profile personal --workspace local
```

**The second command is not optional.** An empty members list means nobody, which is the opposite of how a blank list reads, so the profile reaches no one until they are on it.

Most people start with one profile. More than one is for keeping work and personal credentials, memory and policy apart.

Everything is denied until it is allowed, and the endpoint enforces that when a call is dispatched rather than asking the model to behave:

```
lanes link policy list --profile personal
```

## 4. Connect the accounts

One command per account. It opens a browser and nothing else, and the connection is served straight away with nothing to restart.

```
lanes link connect gmail --workspace local
```

Not sure what a provider needs, or which are already on:

```
lanes link setup plan --workspace local
lanes link setup plan <provider>
```

Memory, tasks, assets, skills, entities and the vault work with nothing connected. They hold the user's own material rather than an account, so there was never anything to authorise.

## 5. Register it with the agents, last

```
lanes link mcp add --workspace local
```

**Register last, after the accounts are connected.** A client reads the tool list when it connects and keeps it, so one registered first holds a list without them until it re-reads. From 0.10.4 that is a delay rather than a dead end: `lanes_tools_search` and `lanes_tools_call` are in every list the endpoint hands out, so an agent can find and invoke an account its own list does not name. Registering last is still the tidier order, but it is no longer something to undo and redo.

With no argument that covers every agent installed on the machine, or name one: `claude`, `codex`. One endpoint, one token, every profile, so it is once per agent rather than once per account. It also installs the usage skill and a scout agent, which is how the agent knows what the endpoint is for; `--no-skill` registers without touching the agent's own files.

Two clients cannot be done this way:

- **Codex** reads the token from the environment rather than its own config. Add to the shell profile: `export LANES_LINK_TOKEN="$(lanes link token show --raw)"`
- **Claude Desktop and Cowork** run the endpoint over stdio from a config file and cannot be given a URL. Send them to https://lanes.sh/docs/link/clients for the JSON block.

Anything else that takes a URL and a bearer token:

```
lanes link outputs --workspace local --show
```

Verify the whole thing:

```
lanes link status --workspace local
lanes link mcp list
```

Then have them start a fresh Claude Code session so the registration loads.

## Critical gotchas

- **Empty members means nobody.** The most common "it connected but sees nothing": a profile with no members list. `lanes link profile members add --me`.
- **A 401 that looks like a bad token is usually `lanes` off the PATH.** The `$(lanes link token show --raw)` substitution silently yields an empty string.
- **`start` and `pair` are independent.** `pair` lets the dashboard read the endpoint. It does not serve it.
- **Register agents last.** A client caches the tool list at connect time. From 0.10.4 an account connected afterwards is still reachable, through `lanes_tools_search`, so this is an ordering preference rather than a re-registration.
- **`--workspace cloud` has its own credential store**, so its token differs from the local one and connections do not carry across.
- **After `lanes link skills add`, the client must reconnect.** A skill is served as a prompt rather than a tool, so no counter moves and nothing announces it.
- **Lanes Link skills are not Claude Code skills.** A Lanes Link skill is one of the user's own procedures, stored per profile and served over the MCP prompts primitive, so the model cannot read a body or pick one; the person invokes it. This SKILL.md, and the one at `~/.claude/skills/lanes-link/SKILL.md`, are the other kind. Do not offer to "add a skill" when someone means either one without saying which you mean.

## Anti-patterns

- Do not connect accounts before deciding the workspace. Migrating them later is real work.
- Do not paste a token anywhere, quote one back, or write one into a file. `lanes link outputs --show` prints it for the person, not for the transcript.
- Do not run `lanes link connect` for a provider they did not ask for, however obvious it seems. Each one is a real OAuth grant.
- Do not restate this setup as a to-do list and stop. Run the commands, check the output of each, and say which step you are on.
- Do not use this skill to *use* the endpoint. Once `mcp add` has run, the tools and the installed `lanes-link` skill take over.

## Then what

- `~/.claude/skills/lanes-link/SKILL.md`, installed by `lanes link mcp add`, is the usage manual: profiles as a boundary, what a refusal means, what to do when it is not running.
- [Quickstart](https://lanes.sh/docs/link/quickstart), [Connect your accounts](https://lanes.sh/docs/link/connect), [Add it to your agent](https://lanes.sh/docs/link/clients), [Deploy to your own cloud](https://lanes.sh/docs/link/deploy).
- Source and licence: https://github.com/lanes-sh/link
