---
layout: post
title: "tmux everywhere"
---

[tmux](http://tmux.sourceforge.net/)/[GNU
Screen](https://www.gnu.org/software/screen/) are a terminal multiplexer. That
means, it re-implements the "Multiple Tabs" features for your console/terminal,
but natively (it works in (almost) any terminal (emulator)) and you can
detach/reattach it (very useful if you just want to leave that script running
for a while without keeping your SSH connection or session opened).

Since I do most of my work remotely, and I don't want to keep opening SSH
connections or I often want to leave everything as-is and come back later, I
don't need to tell you how much I appreciate this.

I also often have unstable internet connections, in which case, I use
[mosh](http://mosh.mit.edu/), an SSH extension which re-establish your
connection when it drops (very useful if you have an unstable internet
connection or if you are moving around with a laptop).

With time, I came up with a small script, to include (or source) at the bottom
your `SHELL`rc/profile to automatically load tmux/screen (it's compatible with
both, prefering tmux). It works with a couple of emulators (edit the checks on
the TERM variable if yours is not part of it), the actual console (text login),
SSH and [mosh](http://mosh.mit.edu/).

You can find it there:
[tmux-everywhere](https://gist.github.com/lemoinem/5670894#file-tmux-everywhere-sh). Along
with my personnal `tmux` config.
