---
layout: post
title: Automatically Starting tmux on SSH
---

It's always a good idea to use [screen][] or [tmux][] when doing any
significant amount of over SSH in case your connection gets dropped.  If you
do this frequently, you might want to have `screen` or `tmux` start
automatically so you don't forget.  To do this, append the following to your
`ssh` command, or if you're using Chrome's [Secure Shell][] extension, add
the following to the "SSH Arguments" field:

    -t -- screen -R

or:

    -t -- /bin/sh -c 'tmux has-session && exec tmux attach || exec tmux'

Explanation:

* `-t` tells SSH to always allocate a pseudo-TTY.  Without this, you'll get a
  `not a terminal` error.

* `--` tells SSH that what follows is a command.  Without this, you'll get a
  cryptic `ssh_exchange_identification: Connection closed by remote host`
  error, at least with Secure Shell.

* `screen -R` attaches to an existing session or creates a new one if none
  exist.

* `tmux` doesn't have an equivalent to `screen`'s `-R` option, so we need to do
  this manually.  The command is mostly self-explanatory, except:

  * `/bin/sh -c` allows us to use `&&` and `||` in our command.

  * `exec` replaces the `/bin/sh` process with `tmux` so that we don't have an
    extra process hanging around.  If you don't care about this, a simple
    `'tmux attach || tmux'` will do.

[Secure Shell]: https://chrome.google.com/webstore/detail/secure-shell/pnhechapfaindjhompbnflcldabbghjo
[tmux]: https://tmux.sourceforge.net
[screen]: https://www.gnu.org/software/screen/
