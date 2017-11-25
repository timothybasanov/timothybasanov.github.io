---
title:  "Marrying Byobu/tmux and Marathono"
---

[Marathono](https://gitlab.com/marathono/marathono) is a great services
manager for macOS. It natively integrates with _launchd_ and works well overall.
Unfortunately due to [#29](https://gitlab.com/marathono/marathono/issues/29)
it always uses login interactive shell with a TTY attached. This triggers
`.zprofile` and friends and [Byobu](http://byobu.co)/tmux interactive mode
and prevents scripts from being executed. There is a simple solution,
detect Marathono in `.zprofile` and skip Byobu/tmux when detected:

```sh
ps -fp $PPID | grep Marathono || \
_byobu_sourced=1 . /usr/local/bin/byobu-launch 2>/dev/null || true
```
