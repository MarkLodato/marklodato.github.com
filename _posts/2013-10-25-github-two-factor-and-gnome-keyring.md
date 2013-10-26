---
layout: post
title: GitHub's Two-Factor Auth and Gnome-Keyring
---

I recently enabled [two-factor authentication][1] on GitHub and then ran into
issues when trying to push with a password.  I usually push via SSH, which is
unaffected by 2FA, but on some machines I don't bother with that and just type
my password in the rare circumstance that I push from that machine.  Sadly,
there is no easy way to use 2FA with this method, so you have to jump through
a few hoops.

First, you need to set up a GitHub [Personal Access Token][2].  This is a long,
random password that avoids the need for a second factor at login. (Google
calls this an "Application Specific Password.")  If you use a Personal Access
Token instead of your regular password when prompted by `git push`, everything
works fine.

However, this creates a second problem: how do you remember that password?  My
solution was to use Gnome Keyring, which recent versions of git support.  It
does not come pre-compiled, so you have to compile and install it yourself
(thanks to [James Ward][3] for these instructions):

```bash
$ sudo apt-get install libgnome-keyring-dev
$ cd /usr/share/doc/git/contrib/credential/gnome-keyring
$ sudo make
$ sudo rm git-credential-gnome-keyring.o
$ sudo mv git-credential-gnome-keyring /usr/bin
$ git config --global credential.helper gnome-keyring
```

Now, so long as you **include your username in the URL**, git will save your
password in Gnome Keyring.  (If you don't put your username in the URL, git
doesn't seem to save the password.)  For example:

```bash
$ git clone https://MarkLodato@github.com/MarkLodato/scripts
```

There is one final caveat.  If you are connected to your workstation via SSH,
you'll need to set up D-Bus, or else you'll get `Error communicating with
gnome-keyring-daemon`.  The solution is to add the following to your
`.bashrc` or `.zshrc` file:

```bash
if [[ -z $DBUS_SESSION_BUS_ADDRESS ]]; then
    if [[ -f ~/.dbus/session-bus/$(dbus-uuidgen --get)-0 ]]; then
        source ~/.dbus/session-bus/$(dbus-uuidgen --get)-0
        export DBUS_SESSION_BUS_ADDRESS
    fi
fi
```

This tells D-Bus to use the instance that was started on the machine's
graphical login, and this in turn allows `git-credential-gnome-keyring` to
talk to the main `gnome-keyring-daemon` instance.  In my case, I have always
unlocked the keyring graphically when I was sitting at the machine, so I have
not yet run into the issue where I have to unlock the keyring from the command
line.

Summary:

1. Create a [Personal Access Token][2] on GitHub and use this as your password.

2. Compile `git-credential-gnome-keyring` from
   `/usr/share/doc/git/contrib/credential/` and add it to your
   `credential.helper` git config.

3. Remember to include `username@` in the push URL, or else git won't save
   your password.

4. If you are SSH'd to your workstation, set up `$DBUS_SESSION_BUS_ADDRESS`.

[1]: https://help.github.com/articles/about-two-factor-authentication
[2]: https://github.com/settings/applications
[3]: http://stackoverflow.com/a/14528360/303425
