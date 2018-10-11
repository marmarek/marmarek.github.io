---
title:  "Qubes OS: Keyboard backlight color based on active qube"
date:   2018-10-11
categories: blog
---

If you have a keyboard with software controlled backlight color,
you can now use it in Qubes OS to show current qube label (and surprise your pets or roommates with amazing technicolor keyboard).

You're going to need a program that will perform the actual change: 
for various Logitech keyboards there is [g810-led]. There's no package
for either Debian or Fedora, but it's quite easy to install manually
(although there's no way to verify source authenticity, and also probably
it didn't get much review, so be careful). Since it's needed only where the
keyboard is connected (it's a USB keyboard, so in sys-usb), it's
enough to install it only there. You can, for example, put the compiled binary in
`/usr/local/bin`. This way, you won't pollute the template on
which sys-usb is based (or have to clone it).

Then, you'll need something that will watch for focus changes in dom0 and
change the keyboard color accordingly. It turned out to be surprisingly easy
with a [simple script][control-keyboard-color.sh]! The script watches for `_NET_ACTIVE_WINDOW` property on the
root window, then follows it (the property holds ID of the active window) and get
`_QUBES_LABEL_COLOR` property for that window's color. The property is
missing for dom0 windows, so I'll default to white backlight there.

Example xprop calls (used in [control-keyboard-color.sh]):

    [user@dom0 ~]$ xprop -notype -spy -root _NET_ACTIVE_WINDOW
    _NET_ACTIVE_WINDOW: window id # 0x4400003, 0x0
    _NET_ACTIVE_WINDOW: window id # 0x0, 0x0
    _NET_ACTIVE_WINDOW: window id # 0x5600859, 0x0
    _NET_ACTIVE_WINDOW: window id # 0x0, 0x0
    _NET_ACTIVE_WINDOW: window id # 0x4400003, 0x0
    ^C
    [user@dom0 ~]$ xprop -notype -id 0x5600859 _QUBES_LABEL_COLOR
    _QUBES_LABEL_COLOR = 13369344
    [user@dom0 ~]$ xprop -notype -id 0x5600859 -f _QUBES_LABEL_COLOR 32x _QUBES_LABEL_COLOR
    _QUBES_LABEL_COLOR = 0xcc0000

When these are in place, you can easily connect them with:

    qvm-run --localcmd=./control-keyboard-color.sh --pass-io \
        sys-usb 'xargs -n1 g810-led -a'

This means: run `.control-keyboard-color.sh` in dom0 and connect it with
`xargs -n1 g810-led -a` running in sys-usb. That xargs command will call
`g810-led -a` for each line in its stdin, with that line appended to the
command. Which is exactly what's needed here, because the line holds the
desired color in hex.
There's one little problem on Qubes 4.0: when using `qvm-run
--localcmd`, the actual command seems to be ignored and all you get is a raw pipe to
`qubes.VMShell` service. That service in practice is just bash, which waits for
commands to execute.  To work around this, [control-keyboard-color.sh] script
prints that command on its first output line.

Now, the only thing left is making all of this start automatically. It's
enough to create `~/.config/autostart/control-keyboard-color.desktop` in dom0, with
the following content:

    [Desktop Entry]
    Name=Control keyboard color
    Exec=qvm-run --localcmd=control-keyboard-color.sh --pass-io sys-usb 'xargs -n1 g810-led -a'
    Terminal=false
    Type=Application

(this assumes the script in somewhere in `$PATH`, for example in `~/bin`)

The above works with USB keyboards connected to sys-usb (or a separate USB
VM if you have the luxury of multiple USB controllers). This in itself
[isn't great from security standpoint][usb-input-security], especially if you connect other
devices to that USB VM. But for many non-laptops these days there is, unfortunately, no
better alternative.

Now, is the time to perform some tests:
<div class='photo-center' style='width: 250px'>
<img  src="/assets/fuzzy-tester.jpg" alt="fuzzy tester"/>
My best fuzzy tester :)
</div>
[g810-led]: https://github.com/MatMoul/g810-led
[control-keyboard-color.sh]: https://gist.github.com/marmarek/2552c395c1b5db56f031936f7dad9156
[usb-input-security]: https://www.qubes-os.org/doc/usb/#security-warning-about-usb-input-devices

