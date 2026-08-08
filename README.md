# herdr-workspace-agents

A live, workspace-scoped agent view for [herdr](https://herdr.dev).

herdr's native agent sidebar shows **every** agent across **every** workspace. With a dozen
workspaces open, that is a wall of panes you have to read past to find the one that wants
you.

This gives you just the current workspace, live.

```
🍋 #1670 lifecycle execution  (wY)  10 agent(s)
  ● wY:p15   working  orch #1670 token-proves-gates-passed
  ◉ wY:p1V   blocked  #1756 pr1757 expired-silent-swallow-deadlines
      ^ waiting on you
  ○ wY:p1T   idle     #1670 pr1752 token-proves-gates-passed
  ✓ wY:pK    done     #1653 pr1660 state-object-rename-rejects-live-uids
```

## Why it is derived rather than requested

There is no server-side way to ask for this. Measured against **herdr 0.7.4**, the socket
API exposes 110 methods and the complete `agent.*` surface is:

```
agent.explain  agent.focus  agent.get   agent.list
agent.read     agent.rename agent.send  agent.start
```

`agent.list` takes `EmptyParams` — no filter argument — and there is no `agent.view.*`
method, nor any view/filter/projection concept anywhere in the schema. So the sidebar cannot
be told to scope itself.

What *does* exist is `events.subscribe`, a real push stream, with `workspace.focused` among
its event kinds. This subscribes to the events that change the answer, re-reads `agent.list`
when one arrives, and prints only the focused workspace. **No polling.**

## Install

```sh
git clone https://github.com/afokapu/herdr-workspace-agents
install -m755 herdr-workspace-agents/herdr-workspace-agents ~/.local/bin/
```

Requires Python 3.10+ and a running herdr. No dependencies.

## Use

```sh
herdr-workspace-agents                 # follow the focused workspace, live
herdr-workspace-agents --workspace wY  # pin to one workspace
herdr-workspace-agents --once          # print once and exit
herdr-workspace-agents --plain         # no ANSI, for piping
```

Set `HERDR_SOCK` if your socket is not at `~/.config/herdr/herdr.sock`.

Run it in a pane and it behaves like a panel — it clears and repaints on each event.

## Status glyphs

| glyph | status | meaning |
|-------|--------|---------|
| ● green | `working` | busy |
| ◉ red | `blocked` | **waiting on you** |
| ○ yellow | `idle` | finished its turn |
| ✓ grey | `done` | session ended |

## Narrow panes keep the identifiers

A worker title is `#<issue> pr<pr> <slug>`. When the pane is too narrow, the **slug** is cut
and the two ids are held back — a third-width pane still answers *which issue, which PR*:

```
 ○ p0   #1671 pr1692 up...
 ● p15  orch #1670 toke...
 ○ p1V  #1756 pr1757 ex...
```

The full pane id (workspace included) and the status are **never** dropped to save room —
they are identifiers you act on. Only the title is cut, because it is the one field you can
still work without.

## Two deliberate choices

**It subscribes only to argument-free event variants.** `pane.agent_status_changed` requires
a `pane_id`, which would mean re-subscribing every time a pane is created. `pane.updated`
covers status changes globally and needs nothing.

**It renders the tab title, not the agent name** — the tab is what the operator named the
work; a worker's own tooling can overwrite its agent name.

**If the workspace cannot be read it shows nothing, and says so** — rather than falling back
to every agent. A view that silently widens when it loses its filter is worse than one that
admits it lost it.

## If it breaks

`events.subscribe` refusing with a missing-field error means the `Subscription` shape
changed. Check which variants still require no arguments:

```sh
herdr api schema --json | python3 -c "
import sys,json
d=json.load(sys.stdin)
..."
```

Verified against herdr 0.7.4 on macOS.

## Licence

MIT
