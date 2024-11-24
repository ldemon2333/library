# Start
## Closing Panes
Closing a pane is as simple as closing a regular terminal session. Either type `exit` or hit `Ctrl-d` and it’s gone.

## Creating Windows
Creating new windows is as easy as typing `C-b c` (one last time: that’s `Ctrl` and `b` at once, then `c`). The new window will then be presented to you in tmux’s status bar.

To switch to the _previous_ window (according to the order in your status bar) use `C-b p`, to switch to the _next_ window use `C-b n`. and `C-b <number>`

## Session Handling
To detach your current session use `C-b d`. You can also use `C-b D` to let tmux give you a choice which of your sessions you want to detach. Detaching from a session will leave everything you’re doing in that session running in the background. You can come back to this session at a later point in time.

To re-attach to a session and continue where you left you need to figure out which session you want to attach to first. List the currently running sessions by using

```
tmux ls
```

This will give you a list of all running sessions, which in our example should be something like

![[Pasted image 20241101152909.png]]

Note that the `-t 0` is the parameter that tells tmux which session to attach to. “0” is the first part of your `tmux ls` output.

If you prefer to give your sessions a more meaningful name (instead of a numerical one starting with 0) you can create your next session using
```
tmux new -s database
```

You could also rename your existing session:
```
tmux rename-session -t 0 database
```
