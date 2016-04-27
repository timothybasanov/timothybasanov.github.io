---
title:  "Configuring byobu/tmux to Support Tabbed/Windowed Terminals"
---

At some point my favorite console "window manager" [byobu](http://byobu.co) switched from *screen* to *tmux* as a backend. *tmux* is much more feature rich and supports splits, for example. 

There is only one downside, but a big one: there could only be one client per *tmux* session. So when you open terminal in several different windows each of them connects to the same *tmux* session and they show exactly the same window. Yes, you can use *tmux* internal windowing capabilities just fine, but I always wanted to have several "real" windows into the same *tmux* session.

{{ excerpt_separator }}

Looks like there is a way to exactly that using *tmux* "session grouping". This piece of functionality is quite quirky and is almost never mentioned in all the articles about *tmux* that I've read. But it does exactly what I need: several sessions looking into the same set of windows.

# Creating several connected sessions in raw *tmux*

As per *tmux* documentation, `new-session` has two parameters of interest:

  - `-s`: Specify session name for a newly created session
  - `-t`: Link newly created session to a different session creating a group

You have to name your sessions to specify that name later in a `-t` parameter.

```sh
terminal1$ tmux new-session -s main \; new-window
terminal2$ tmux new-session -s secondary -t main \; new-window
# Both terminals now share the same set of windows,
# but different windows are active
```

# Use one persistent session and make others temporary

*tmux* configuration language is quite restricted. I ended up with having one session that is persistent and is never destroyed or attached to and all the others live in a "create, attach, work, detach & destroy" lifecycle. Generally I have *N+1* sessions when *N* clients are connected.

  - `tmux new-session -d -s shared`: Create shared session, but never attach to it
  - `tmux set-option -t shared destroy-unattached off`: Set an option for that session to make it persistent
  - `tmux set-option -g destroy-unattached on`: Set a global default to make sessions destructible (we set an option for the *shared* session to overwrite this default)

Now when new client session is created you can connect to a *shared* one: `tmux new-session -t shared`. This session will be destroyed on a terminal closure.

# Convince *byobu* to use our new configuration

To make *byobu* read your configuration it's sufficient to add this to your `~/.byobu/.tmux.conf` configuration file:

```sh
new-session -d -s shared
set-option -t shared destroy-unattached off
set-option -g destroy-unattached on
```

> Assuming you use *homebrew* main *byobu* file is in `/usr/local/bin/byobu` which always sources `/usr/local/share/byobu/profiles/tmuxrc` which always sources `~/.byobu/.tmux.conf`.

As now you have many sessions instead of one, which *byobu* expects, on each new window you'll see a `byobu-select-session` prompt. Moreover, new sessions would not be grouped with our special *shared* one.

There are three ways to convince byobu to skip `byobu-select-session`:

  - Pass additional command line parameters to `byobu` when calling from the command line.
  - Set an environment variable `BYOBU_WINDOWS` before calling *byobu* and create `~/.byobu/window.tmux.$BYOBU_WINDOWS` non-empty file
  - Create `~/.byobu/window.tmux` non-empty file with exactly one line of *tmux* configuration.

I chose the last one as it's always enabled and does not require fiddling with command line parameters or environment variables.

So, you can add this to your `~/.byobu/window.tmux` to create new sessions always attached to your *shared* session.

```sh
new-session -t shared
```

That's it. Two files, four lines of code and your *byobu* now works well with dumb terminals (not supporting *tmux* natively) opening several windows.

Good luck!

# Full text of my current *byobu/tmux* configuration

<script src="https://gist.github.com/timothybasanov/00f109853d73135749ccd4884312bcb0.js"></script>
