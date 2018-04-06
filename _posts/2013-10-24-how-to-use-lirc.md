---
layout: post
title: Controlling Linux with a Remote Control
---

I listen to [Pandora](http://pandora.com) all the time at home, and I often
want to control it without using the keyboard or mouse.  So, a few years ago I
bought a USB infrared receiver for $14.  Of course, since I use Linux, the
hidden cost was hours and hours of time figuring out how to set it up.  I
never wrote anything down back then, but since I had to relearn it all last
weekend to make a few changes, now is a good time to document it.

<!-- more -->

This article is written for Ubuntu 13.04, so some things might be different
for other operating systems.  It also assumes you want to use an infrared
remote; you could also use a bluetooth remote or some cellphone app, but I
don't know anything about those.

### Summary ###

1. Buy a "USB Media Center IR receiver" (plus a remote if you don't have one.)
2. If you have an MCE *remote*, disable the keyboard emulation in X.
3. Run `irrecord` and follow the instructions to generate a remote definition
   file.
4. Set up `/etc/lirc/` to use the new remote definition file.
5. Restart LIRC and test with `irw`.
6. Either (a) configure `/etc/lirc/lircrc`, or (b) configure `~/.lircrc` and
   set up `irexec` to start at session startup.

### Hardware ###

#### Buying the IR reciever ####

First, you need an IR receiver.  The safest bet is to buy a "USB Media Center
IR receiver," aka an "MCE receiver." I got the following one for $14,
including shipping:

![My IR Receiver]({{ site.url }}/images/2013-10-24-ir-receiver.jpg)

Note that [IrDA](http://en.wikipedia.org/wiki/Infrared_Data_Association) ports
found in older laptops won't work for this purpose.

#### Choosing a remote ####

Second, you need an IR remote.  You could use an existing remote if you know
it won't cause a conflict---say, you no longer use the TV it goes with, or the
TV and the computer are in different rooms.  Or, you could buy a new remote
just for the computer.  But the most likely option is that you have an
existing remote that has a "universal" feature.  In that case, all you need to
do is configure the universal feature to use some non-existent device.  For
example, if your TV remote can control a DVD player, and you don't have DVD
player, configure the remote to control any DVD player listed in the remote's
manual.  It doesn't matter which one---LIRC can (probably) handle any.

### Configuring X (if you have an MCE remote) ###

If you have a Windows Media Center (MCE) *remote* (as opposed to, say, a TV or
DVD remote), or if you have set up your universal remote to send MCE codes,
then X will automatically interpret some buttons as keyboard keys.  If this is
all you want, then you're done!

Unfortunately, most people will want more than this, in which case the
built-in keyboard emulation will just get in the way.  So, you need to disable
it.  If you forget to do so, X will send the keyboard keys in addition to
whatever you configure LIRC to do, and it will appear that you're getting
double key presses.

First, you need to determine the product name for your IR receiver.  To do
this, make sure your IR receiver is plugged in and look through
`/proc/bus/input/devices` to find the name of your device.  Here's the output
on my computer:

```bash
$ grep '^N:' /proc/bus/input/devices
N: Name="Power Button"
N: Name="Power Button"
N: Name="Microsoft Comfort Curve Keyboard 2000"
N: Name="Microsoft Comfort Curve Keyboard 2000"
N: Name="Logitech Unifying Device. Wireless PID:101b"
N: Name="Media Center Ed. eHome Infrared Remote Transceiver (1934:5168)"
N: Name="MCE IR Keyboard/Mouse (mceusb)"
```

Find the one that sounds like "Media Center ... Remote Transceiver."  Then,
create the file `/usr/share/X11/xorg.conf.d/10-mce-keyboard-disable.conf` with
the following contents, replacing the value of `MatchProduct` with whatever
name you found above.  (In the old days, we would have added this to
`/etc/X11/xorg.conf`, but now evidently we create these xorg.conf.d files.)

```bash
# Disable the MCE remote from acting like a keyboard.  (We use lirc instead.)
Section "InputClass"
    Identifier   "MCE USB Keyboard mimic blacklist"
    Driver       "mceusb"
    MatchProduct "Media Center Ed. eHome Infrared Remote Transceiver (1934:5168)"
    Option       "Ignore" "on"
EndSection
```

### Setting up LIRC ###

[LIRC](http://www.lirc.orc) is the program that interprets IR signals.  To
install it, run:

```bash
$ sudo apt-get install lirc lirc-x
```

LIRC works in two steps: the `lircd` program that translates remote signals to
generic "buttons," and then client programs---notably `irexec`---translate
buttons to actions.

So, three are things you need to do:

1. Create a remote definition file, unless one already exists.
2. Tell `lircd` to use the proper definition file.
3. Configure `irexec`.

If you're using a non-MCE receiver, you may also have to spend some time
configuring LIRC to use your receiver, but thankfully MCE receivers work out
of the box.

#### Training LIRC ####

Unless there already happens to be a file in `/usr/share/lirc/remotes/` that
works with your remote, you are going to have to train LIRC to understand all
the buttons on your remote.  (If there is one already, skip to the next
section.)

Make sure that `lircd` is not already running, and then use the `irrecord`
program to create what I am calling a remote definition file.  Obviously,
replace `WHATEVER_YOU_WANT` with, well, whatever you want to call your remote.

```bash
$ sudo service lirc stop
$ sudo irrecord --device=/dev/lirc0 /etc/lirc/lircd.WHATEVER_YOU_WANT.conf
```

Follow the on-screen instructions, and when it when it asks you to "Please
enter the name for the next button," open up a new terminal window and run

```bash
$ irrecord --list | less
```

to find an appropriate name for the button you want to learn.  I recommend
only programming four or five buttons for now, and then moving on to the next
section.  I ended up having to delete my config and start over several times,
so it's not worth investing too much time now.

Once you have tested your config with `irw` in the next section and are
confident it is working, you can run the same `irrecord` command again to add
more buttons.  As long as you point it to the same config file, you won't have
to go through the training step again.

#### Configuring lircd and testing with irw ####

Now we have to tell `lircd` to use the new definition file.  We do this in two
places:

First, in `/etc/lirc/lircd.conf`, `include` your remote definition file.  This
will be the only command in the file.

```bash
include "/etc/lirc/lircd.WHATEVER_YOU_WANT.conf"
```

Second, in `/etc/lirc/hardware.conf`, set `REMOTE_LIRCD_CONF` to the remote
definition file.  (I don't think this does anything, but it doesn't hurt.)

```bash
REMOTE_LIRCD_CONF="/etc/lirc/lircd.WHATEVER_YOU_WANT.conf"
```

Now, start `lircd` and test it with `irw`.  Press a few of the buttons you
have trained in the previous section and make sure they show up in `irw`.  For
example:

```bash
$ sudo service lirc start
$ irw
000000037ff07bed 00 KEY_CHANNELUP mce_custom
000000037ff07bec 00 KEY_CHANNELDOWN mce_custom
```

If you get one line per press (plus repeated lines if you hold down the
button), it works!  Feel free to go back to the previous section and configure
more buttons.

#### Configuring irexec via lircrc ####

The last step is to configure `irexec`, which is the thing that actually does
something useful when you press a button.  You have two choices here:

A. Create `/etc/lirc/lircrc` and have `irexec` run as root.  This has the
   advantage of starting and stopping `irexec` automatically whenever you
   start/stop `lircd`.

B. Create `~/.lircrc` and have `irexec` runs as your user.  If you do this,
   you need to configure your desktop environment to automatically start
   `irexec` at the beginning of each session.  You will also need to restart
   `irexec` every time `lircd` restarts.

Create one of the two files listed above.  The
[documentation](http://www.lirc.org/html/configure.html#lircrc_format) is
pretty good, but here is my config file for reference:

```
begin
    button = KEY_VOLUMEUP
    prog = irexec
    config = /usr/bin/pactl -- set-sink-volume 0 +2%  # vol+
    repeat = 1
end

begin
    button = KEY_VOLUMEDOWN
    prog = irexec
    config = /usr/bin/pactl -- set-sink-volume 0 -2%  # vol-
    repeat = 1
end

begin
    button = KEY_PLAY
    prog = irexec
    config = /home/mark/local/bin/pithos-control play
end

begin
    button = KEY_PAUSE
    prog = irexec
    config = /home/mark/local/bin/pithos-control pause
end

begin
    button = KEY_STOP
    prog = irexec
    config = /home/mark/local/bin/pithos-control pause
end

begin
    button = KEY_FASTFORWARD
    prog = irexec
    config = /home/mark/local/bin/pithos-control skip
end

begin
    button = KEY_SLEEP
    prog = irexec
    config = /usr/bin/xset dpms force off  # blank monitor
end
```

The file is pretty self-explanatory. Each `button` is something I defined in
my remote definition file.  The `pactl` program works with PulseAudio, which
is the default audio system on Ubuntu.  The `pithos-control` program is
described in the next section.  And the `xset` line turns off my monitor,
which I often want to do if I'm listening to music.

When you're done, remember to either `sudo service lirc restart` if you are
using `/etc/lirc/lircrc` or `irexec -d` if you are using `~/.lircrc`.
Otherwise, nothing will happen when you press a button.

### Aside: My Pithos control script ###

I always use [Pithos](http://kevinmehall.net/p/pithos/) to play Pandora.  Not
only is it *way* lighter than the bloated web-based one, it is also
controllable via D-Bus.  Sending D-Bus commands manually is a bit ugly, so I
made a little wrapped called
[pithos-control](https://github.com/MarkLodato/scripts/blob/master/pithos-control).
The main advantage is that it adds discrete play and pause commands, which
Pithos lacks.

### Appendix: (Not) Using ir-keytable ###

When I was trying to set up my remote, I found plenty of suggestions on the
Internet to use something called `ir-keytable` instead of, or in conjunction
with, LIRC.  I did use it for a while, but in the end I found that using
straight LIRC was way better.

The main problem was that `ir-keytable` (by itself) creates virtual keyboard
presses for each button press.  If you want to control, say, the volume, you
would configure the "Vol-" button to send the XF86AudioLowerVolume key.  This
sounds fine, but the issue is that, at least in XFCE, the media keys cause the
foreground window to lose focus each time they are pressed.  So, if I were
typing something and my wife hit the volume button, a few of my keystrokes
would be lost, which was really irritating.  It was even worse when I was
playing a game.

I put up with the above for a long time, but the final straw was when I tried
to get the `xset dpms force off` command to work.  It was impossible with
`ir-keytable` because the virtual keyboard button that I had to map the
command to would immediately wake the monitor up.  This ended up being a
blessing in disguise, since it made me realize that I could ditch
`ir-keytable` for LIRC.

I also found that `ir-keytable` was less reliable than LIRC.  I have no hard
evidence of this and I am not inclined to set it up again, but that was my
impression.

### Update: Ubuntu 17.04 breaks lirc ###

My setup stopped working after upgrading to Ubuntu 17.04. It appears this is
a very common problem. When I ran `irw` it showed nothing, and when I ran
`mode2` I got:

```bash
$ mode2 -d /dev/lirc0
Using driver devinput on device /dev/lirc0
Trying device: /dev/lirc0
Using device: /dev/lirc0
Running as regular user mark
Partial read 16 bytes on /dev/lirc0
```

The solution, for me, was to set the following option in
`/etc/lirc/lirc_options.conf`:

```conf
driver = default  # was devinput
```
