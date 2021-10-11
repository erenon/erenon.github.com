---
title: "Input Latency"
date: 2021-10-11
---

This post was inspired by [Dan Luu: Computer Latency][0].
I felt that my work setup (see below) is sluggish, even though I got used to it.
I wanted to quantify how sluggish it really is.
Also, VSCode was advertised to offer a lower latency experience (compared to ssh+vim),
as it runs and renders on the local machine: I didn't feel it - I wanted to verify my observation,
knowing that I use vim for too long to acknowledge its shortcomings.

## Results

Program    | Latency (ms)
:--------|------:
kitty     | 90
gvim   | 100
vscode | 140
remote<sup>2</sup> vim | 195
remote vscode | 200
remote Visual Studio | 210

These are the test results of the latency between a keypress and the display of a character (see below for details).
Results are rounded to the nearest `5 ms` to avoid the impression of false precision.

The measurement confirms that typing on this particular remote setup is twice as bad as locally,
and that the additional latency of vscode (compared to vim) is greater than the latency of ssh.

## Experimental setup

A local computer was filmed with a 240 FPS camera.
Latency is defined as the time (number of frames) elapsed between a keyboard key starts moving,
and the matching character appears on screen. The declared latency is a rounded average of several
key presses, without much deviation. No particular care was taken to achieve the lowest latency possible
(e.g: CPU pinning, interrupt pinning, reducing workload) -- as that's not how I normally work --
but the computers used were under no particular load in general.

The **local computer** runs on a AMD FX 8320, with a standard HP KU-0316 keyboard attached via USB,
and a 4K 60Hz AOC monitor attached via DisplayPort. The operating system is Linux 5.14.7,
running the latest software from the ArchLinux distribution, including the i3 window manager.

The **remote computer** is a dedicated machine, hosted in a datacenter, ~1500 kilometers away from
the local computer, with a modern Intel CPU and plenty of RAM. It is accessed with xfreerdp,
and runs Windows 10.

The **remote server** is a server class Intel machine of unspecified details, runs Linux 3,
and somewhat older versions of software. Regarding physical location, it is close to the remote computer described above.
It is shared between different users, but during the test, no particular load was observed.

## Contenders

[kitty][] is a powerful terminal emulator, version 0.23.1 was used.
Runs on the **local computer**.

[gvim][vim] is the GTK GUI version of the venerated editor, version 8.2.
Runs on the **local computer**.

[vscode][] is a recent Microsoft product, version 1.60.2.
Runs on the **local computer**.

remote [vscode][] runs on the **remote computer**, version 1.53.2.

remote [Visual Studio][] runs on the **remote computer**, version 16.9.2.

remote<sup>2</sup> [vim][] is version 8.2. It runs in a tmux instance on the **remote server**,
accessed via PuTTY, that runs on the **remote computer** (that is accessed via xfreerdp, from the **local computer**),
totalling to 2 logical hops in one direction.
Contorted? Yes. Unrealistic? Well, that's how I have to work...



[0]: https://danluu.com/input-lag/
[kitty]: https://sw.kovidgoyal.net/kitty/
[vim]: https://www.vim.org/
[vscode]: https://code.visualstudio.com/
[Visual Studio]: https://visualstudio.com/
