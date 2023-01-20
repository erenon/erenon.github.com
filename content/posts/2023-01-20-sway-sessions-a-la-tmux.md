---
title: "Sway sessions à la tmux"
date: 2023-01-20
---

A window manager allows the user to organize windows of running programs
on a desktop. Desktops are mapped to outputs (i.e: monitors).
Most window managers support [Virtual Desktops][], allowing the user
to move windows to separate desktops and quickly switch which desktop is shown on a given output
(how quickly depends on the window manager: on popular commercial systems, not so quickly).

[Virtual Desktops]: https://en.wikipedia.org/wiki/Virtual_desktop

The used workspaces might be shown to the user in a taskbar-like fashion, e.g: on [Sway][]:

```
[1] [2] [3]
```

[Sway]: https://swaywm.org/

Virtual desktops (or workspaces, in the [i3][]/sway terminology) enlarge the available screen estate.
By putting certain windows on well-known workspaces, one can quickly find them without cycling
through all the windows by <kbd>Alt</kbd>+<kbd>Tab</kbd>-ing (or [even](https://wanderingstan.com/wp-content/uploads/2009/07/alt-tab-flowcharts-labeled-2.png)...).
For example, I always put a browser to workspace 1, an editor to workspace 2, and a shell to workspace 3.
This way I can always quickly jump to any of them, by pressing <kbd>Mod</kbd>+<kbd>1/2/3</kbd>.

[i3]: https://i3wm.org/

What happens if I have to work on two things (_foo_ and _bar_) concurrently? I can open 3 more workspaces:

```
[1] [2] [3] [4] [5] [6]
```

Now this is a bit unfortunate. If the workload is dynamic enough, I'll have a hard time
quickly telling what is running where. It can be improved by renaming some workspaces, e.g:

```shell
$ swaymsg rename workspace 1 to 1:foo
$ swaymsg rename workspace 4 to 4:bar
```

So it'll look like this:

```
[1:foo] [2] [3] [4:bar] [5] [6]
```

While it is certainly better, it isn't ideal. Muscle memory doesn't help any more: while editing the second project
(workspace 5), pressing the usual <kbd>Mod</kbd>+<kbd>3</kbd> will jump to the shell of the first
project (workspace 3), not to workspace 6, that hosts the shell of the second project.

Also, if I want to add a new workspace to the working set of _foo_... I can't,
without shifting subsequent workspaces to the right first.

Furthermore, pressing <kbd>Mod</kbd>+<kbd>6</kbd> with left hand only is not comfortable to me.

The famous [tmux][] terminal multiplexer offers a solution: it calls them _sessions_.
Each session holds a distinct set of workspaces (windows, in the tmux terminology).
Sessions can be named and easily switched to. Only the workspaces of the selected session
are shown, and pressing the right combination selects the numbered workspace of the active session.
This is (almost) exactly what I wanted. Sway does not provide this functionality out of the box,
so I created some scripts to emulate it. Let's see:

[tmux]: https://github.com/tmux/tmux

## Scripts to emulate sessions

First, there's `sway_rename_workspace.py`, that simply wraps `swaymsg rename workspace`:

```python
#!/usr/bin/python3

# Prompts for a name and renames the current workspace to it, ESC cancels.
# It preserves the number of the workspace - therefore the ordering.

import json
import subprocess

def ui_input(prompt):
    s = subprocess.check_output(["/usr/bin/dmenu", "-b", "-p", prompt], input="", encoding="ascii")
    return s.strip()

def focused_workspace_num():
    workspaces = subprocess.check_output(["/usr/bin/swaymsg", "-r", "-t", "get_workspaces"])
    workspaces = json.loads(workspaces)
    for ws in workspaces:
        if ws["focused"]:
            return ws["num"]
    return ""

name = ui_input("rename workspace")
num = focused_workspace_num()

subprocess.check_output(["/usr/bin/swaymsg", "rename", "workspace", "to", f"{num}:{name}"])
```

It prompts for a name using [dmenu][]. Perhaps dmenu is not the most precise choice,
being an [X][] application, but it was already installed and perfectly matches my use-case.

[dmenu]: https://tools.suckless.org/dmenu/
[X]: https://www.x.org/wiki/

Second, there's `sway_select_session.py`, slightly more than a one-liner:

```python
#!/usr/bin/python3

# Prompts for a workspace name, ESC cancels.
# If a name is given, it looks for the selected workspace.
# If a workspace is found, it moves that workspace, and every
# workspace to the right -- until a different named-workspace is found --
# to the beginning, and moves every other workspace to the end,
# starting at workspace number 10.
# If no workspace matches the given name, it will be created,
# and every other workspace will be moved to the end.
# It does not touch workspaces given in `protected_workspaces`.

import json
import subprocess

# do not move these workspaces
protected_workspaces = set([1, 9])

def ui_input(prompt, selection):
    s = subprocess.check_output(["/usr/bin/dmenu", "-b", "-p", prompt], input=selection, encoding="ascii")
    return s.strip()

def strip_before(s, d):
    a,_,b = s.partition(d)
    return b or a

def selected_range(workspaces, s):
    first = 0
    for ws in workspaces:
        if ws["name"] == s:
            break
        first += 1
    last = first + 1
    for ws in workspaces[last:]:
        if ':' in ws["name"]:
            break
        last += 1
    return first, last

def renumber(i, ws):
    tag = ws["name"].partition(":")[2]
    return f"{i}:{tag}" if tag else str(i)

# select a session, find the range of workspaces that belongs to it
workspaces = subprocess.check_output(["/usr/bin/swaymsg", "-r", "-t", "get_workspaces"])
workspaces = json.loads(workspaces)

sessions = []
for ws in workspaces:
    session = ws["name"].partition(":")[2]
    if session:
        sessions.append(ws["name"])

selected_session = ui_input("select session", "\n".join(sessions))
first, last = selected_range(workspaces, selected_session)

# move selected worspaces to si..., deselected workspaces to di...
si = 1
di = max(last-first, 10)

renames = []

for (i, ws) in enumerate(workspaces):
    if ws["num"] in protected_workspaces:
        continue
    if first <= i and i < last:
        while si in protected_workspaces:
            si += 1
        renames.append((ws["name"], renumber(si, ws)))
        si += 1
    else:
        while di in protected_workspaces:
            di += 1
        if len(renames) == 0 and not ":" in ws["name"]:
            # first deselected workspace is unnamed, name it "main" to avoid
            # merging it with the selected session
            renames.append((ws["name"], str(di) + ":main"))
        else:
            renames.append((ws["name"], renumber(di, ws)))
        di += 1

# move them to tmp names to avoid clashes
for (src, dst) in renames:
    subprocess.call(["/usr/bin/swaymsg", "rename", "workspace", src, "to", f"_{dst}"])

# move them back
for (_, dst) in renames:
    subprocess.call(["/usr/bin/swaymsg", "rename", "workspace", f"_{dst}", "to", dst])

# finally, jump to the leading workspace if the selected session
# (this also creates the session if it is a new one)
tag = strip_before(selected_session, ":")
i = 1
while i in protected_workspaces:
    i += 1

subprocess.call(["/usr/bin/swaymsg", "--", "workspace", "--no-auto-back-and-forth", f"{i}:{tag}"])
```

So it prompts for a name (a session name), identifies the workspaces that belong to that session,
moves them to the beginning, and moves the others to the end. Let's see an example. We start with:

```
[1:foo] [2] [3] [4:bar] [5] [6]
```

Then when we select "4:bar", we'll end up with:

```
[1:bar] [2] [3] [10:foo] [11] [12]
```

...where "1:bar" was "4:bar", "2" was "5", "3" was "6", "10:foo" was "1:foo", "11" was "2" and "12" was "3".
This new layout allows muscle memory to kick in, keeps frequently used workspace numbers small,
and also adds a gap between the first and the other sessions: if needed, workspace 4 can be quickly added to
the _bar_ session.

A personal twist appears here: 1 and 9 are `protected_workspaces`, virtually part
of every every session -- I never want to move them. I keep there communication tools,
a journal, a music player, etc.

Finally, if you want to stop _sessioning_, there's `sway_flatten_sessions.py`:

```python
#!/usr/bin/python3

# Assign each workspace the lowest number still available.

import json
import subprocess

# do not move these workspaces
protected_workspaces = set([9])

renames = []
index = 1

workspaces = subprocess.check_output(["/usr/bin/swaymsg", "-r", "-t", "get_workspaces"])
workspaces = json.loads(workspaces)
for ws in workspaces:
    if ws["num"] in protected_workspaces:
        continue
    session = ws["name"].partition(":")[2]
    if session:
        renames.append((ws["name"], f"{index}:{session}"))
    else:
        renames.append((ws["name"], f"{index}"))
    index = index + 1
    while index in protected_workspaces:
        index = index + 1

for rn in renames:
    subprocess.call(["/usr/bin/swaymsg", "rename", "workspace", rn[0], "to", rn[1]])
```

This simply renumbers the workspaces in order, putting them next to each other, removing any gap.
To tie them together, I'm using the following sway piece of config:

```
set $mode_session session [s]elect [r]ename [f]latten
bindsym $mod+s mode "$mode_session"
mode "$mode_session" {
    bindsym s exec --no-startup-id ~/bin/way_select_session.py, mode "default"
    bindsym r exec --no-startup-id ~/bin/way_rename_workspace.py, mode "default"
    bindsym f exec --no-startup-id ~/bin/way_flatten_sessions.py, mode "default"

    bindsym Escape mode "default"
    bindsym Return mode "default"
}
```

This allows very quick session renaming (<kbd>Mod</kbd>+<kbd>s</kbd><kbd>r</kbd>) and selection (<kbd>Mod</kbd>+<kbd>s</kbd><kbd>s</kbd>).
The presented code is also available on [GitHub](https://gist.github.com/erenon/024679fecf849454f54d245537f3a783).

