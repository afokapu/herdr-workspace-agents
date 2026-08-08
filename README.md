# herdr-navigator

A live, workspace-scoped agent view for [herdr](https://herdr.dev).

herdr's native agent sidebar shows **every** agent across **every** workspace. With a dozen
workspaces open, that is a wall of panes you have to read past to find the one that wants
you.

This gives you just the current workspace, live.

```
🍋 #1670 lifecycle execution  (wY)  13 tabs
>● 1  wY:t1   working orch #1670 token-proves-gates-passed
 ◉ 18 wY:tJ   blocked #1756 pr1757 expired-silent-swallow-deadlines
      ^ waiting on you
 ○ 19 wY:tK   idle    #1670 pr1752 token-proves-gates-passed
 ✓ 17 wY:tH   done    #1653 pr1660 state-object-rename-rejects-live-uids
```

`>` marks the focused tab. The leading number is the tab number — what you press to switch
to it — so it is never dropped.

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
install -m755 herdr-workspace-agents/herdr-navigator ~/.local/bin/
```

Requires Python 3.10+ and a running herdr. No dependencies.

## Use

```sh
herdr-navigator                 # follow the focused workspace, live
herdr-navigator --workspace wY  # pin to one workspace
herdr-navigator --once          # print once and exit
herdr-navigator --plain         # no ANSI, for piping
```

Set `HERDR_SOCK` if your socket is not at `~/.config/herdr/herdr.sock`.

Run it in a pane and it behaves like a panel — it clears and repaints on each event.

## Keys

```
j / k               move  (arrow keys never arrive — see below)
enter               switch to the selected tab or pane
click a row         switch to it
click off the list  dismiss
/                   search — type to filter, enter to keep, esc to clear
[ / ]               fold / unfold the tab (from a pane row, its parent tab)
space               toggle fold
z                   fold everything, or unfold everything
b w d a             filter by state: blocked, working, done, all
n / s / t           toggle the id, status and title columns
esc / q             close
```

### Why not the arrow keys

herdr binds them itself, and its config says so without qualification:

```
# navigate_pane_left = "h"      # left arrow always focuses the pane to the left
# navigate_pane_right = "l"     # right arrow always focuses the pane to the right
# navigate_workspace_up = "up"
# navigate_workspace_down = "down"
```

They never reach a pane app. In a session-modal popup, pressing one moves herdr's focus
away and the popup closes — which reads as "the arrow key dismissed it". `j`/`k` move,
`[`/`]` fold.

`h`/`l` are avoided for the same reason, and `i` is left alone because herdr's own agent
filters use it.

## Summary panel

Above the tree, what needs you before where it is:

```
┌──────────────────────────────────────────────────────────────────┐
│ #1670 lifecycle execution                                    wY  │
│                                                                  │
│ ● 0           ◆ 1           ○ 6           ✓ 2           · 3      │
│ blocked       working       idle          done          other    │
│                                                                  │
│▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒░░░░░░░░░············________ │
│ 12 panes in 11 tabs                                              │
└──────────────────────────────────────────────────────────────────┘
```

The count sits above its label so the eye lands on the number. `blocked` is bold when
non-zero — it is the only state that wants something from you.

The bar is one cell per pane. Reading a mix is faster than reading five numbers, and the
mix is what says whether a workspace is busy, stalled, or waiting on you. Each state has
its own fill (`█ ▓ ▒ ░ ·`) as well as its own colour, so the bar still reads when piped to
a file — colour alone would make `--plain` a single meaningless block.

Counted over **panes**, not tabs: a tab's status is an aggregate, so counting tabs would
under-report a workspace where one tab holds three agents. The parts **sum to the pane
count**, asserted in code — `other` exists because shell panes report `unknown`, and a
tally that dropped what it did not recognise would show 9 of 12 and look complete. Zeroes
are dimmed rather than hidden: an absent count reads as "not measured".

## What it shows

A tab→pane tree for the current workspace, mirroring herdr's own navigator:

```
🍋 #1670 lifecycle execution  (wY)
→◆ 1  wY:t1   orch #1670 token-proves-gates-passed             1 panes
 ◆├─◆ p15  orch #1670 token-proves-gates-passed        claude · working
 ○ 17 wY:tH   #1653 pr1660 state-object-rename-rejects...      1 panes
  ├─○ pK   #1653 pr1660 state-object-rename-rejects...   claude · idle
```

`▾` / `▸` show whether a tab is expanded; a tab with no panes gets neither, because a
triangle that does nothing misdescribes what the key will do. `→` marks the focused tab,
`◆` the focused pane. The right margin carries the pane count
for a tab and `agent · status` for a pane. Searching keeps a tab whose *panes* match, so
filtering never orphans a result from its parent.

## Status glyphs

| glyph | status | meaning |
|-------|--------|---------|
| ● green | `working` | busy |
| ? magenta | `unknown` | a tab with no live agent |
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
