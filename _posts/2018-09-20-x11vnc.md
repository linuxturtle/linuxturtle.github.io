---
title: x11vnc and xtightvncviewer r0x0r
date: Sep 20, 2018 10:43:30 AM
layout: post
---

I've been playing around a bit with 
<a href="http://www.karlrunge.com/x11vnc/">x11vnc</a> and
<a href="http://www.tightvnc.com/">tightvnc</a>, and I've concluded that
they both seriously rock.  They are enabling me to do stuff I've wanted to do
for years, but thought wasn't reasonably possible.

<b>Desktop Sharing</b>

I occasionally want to share something on my desktop with
other people, or visa-versa -- generally to facilitate remote collaboration with them.
People that use a certain evil <a href="http://microsoft.com"><b>other</b> OS</a>
often want to use a 
<a href="http://www.microsoft.com/windows/netmeeting/">nasty proprietary tool</a>
to accomplish the task, since they don't know any better -- but being a linux
geek, and someone that values his freedom, and has an intense dislike of
proprietary closed software, that's not an option for me :) 

Now, I've been a fan of <a href="http://www.realvnc.com/">VNC</a> pretty much since
its inception at AT&T labs.  It is very cool technology, and it allows for
sharing a desktop between platforms.  But the original VNC was only capable of
generating VNC protocol from its own X-ish sessions, or from a windoze desktop
-- i.e. I could create a virtual X session that multiple people could share, but
I couldn't share my everyday X display, where I'm most comfortable, and where I'm
likely to be working when the urge to share something comes up :).

Enter x11vnc.

x11vnc is a fork of the VNC code, which allows me to attach the VNC protocol
generation to my current X session.  Not only that, but in combination with some
of the cool stuff the tightvnc project has been developing, makes it <b>extremely
easy</b> to share desktops between platforms.

On the windows side, tightvnc is very easy to set up, and well 
<a href="http://www.tightvnc.com/winst.html">documented</a>.  On the *nix side,
similar functionality requires two seperate packages (x11vnc and tightvnc), and
the documentation for doing so is, umm, well, not very easy to find.  So, without
further ado, here's a very simple desktop sharing script:

<pre>
    #!/bin/sh
    pkill x11vnc
      
    VNC_JAVA_DIR="/usr/share/tightvnc-java"
    HTTP_PORT=5999
    VNC_COMMAND="x11vnc -display :0 -viewonly -httpdir $VNC_JAVA_DIR \
	     -httpport $HTTP_PORT -shared -nothreads -bg -alwaysshared \
	     -dontdisconnect -many -cursor"
      
    port="$($VNC_COMMAND | sed -e 's/PORT=//')"
    port=$(expr $port - 5900)
      
    zenity --display :0 --info \
	   --text="x11vnc is running on VNC display $port.  Click OK to kill"
      
    pkill x11vnc
</pre>

This little script will share your desktop in view-only mode, until you click
"OK" on the dialog box it pops up.  It depends on having the "xllvnc", "zenity",
and "tightvnc-java" packages installed on debian/ubuntu -- other distros may
have different names, but you get the idea.  You can use a vnc viewer to connect
to the shared desktop, or you can simply point a web browser at the shared
machine on port 5999 (i.e. http://&lt;shared-machine-name&gt;:5999), and you'll get a
java applet that displays the shared desktop inside the browser.  Pretty cool :)

Personally, on my desktop machine, I have a pretty large desktop, and don't
really want to share the whole thing.  So I've taken the above script a little
further, and shared only a portion of my desktop -- then I start up a xlogo
window on that part of my desktop, so I know where it is.  Now, any window I
drag into that area is automatically shared (and I can still read email, or type
in IRC, or whatever on the rest of my desktop ;).  Here's my actual sharing script:

<pre>
    #!/bin/sh
    killall x11vnc
    killall xlogo
      
    SHARE_GEOMETRY="1024x768+2176+432"
    VNC_JAVA_DIR="/usr/share/tightvnc-java"
    HTTP_PORT=5999
    VNC_COMMAND="x11vnc -display :0 -viewonly -httpdir $VNC_JAVA_DIR \
	     -httpport $HTTP_PORT -clip $SHARE_GEOMETRY -xinerama \
	     -shared -nothreads -bg -alwaysshared -dontdisconnect -many -cursor"
      
    xlogo -geometry $SHARE_GEOMETRY&
      
    port="$($VNC_COMMAND | sed -e 's/PORT=//')"
    port=$(expr $port - 5900)
      
    zenity --display :0 --info \
	   --text="x11vnc is running on VNC display $port.  Click OK to kill"
      
    killall x11vnc
    killall xlogo
</pre>

<hr>
<b>Accessing My Desktop Remotely</b>

The other problem I've been wanting to solve for a long time involves accessing
the desktop running on a machine I don't have physical access to.  For example,
I have occasionally found myself at home, but wanting access to an X application I
left running on my workstation in my cube at work.  I could run x11vnc all the
time, on all machines I might possibly want to access, but that's a painful
thing to maintain, and it's rather insecure as well.

Again, enter xllvnc and tightvnc...

It turns out that the tightvnc viewer has this really neet "-via" option, which
lets you access a VNC session through a ssh tunnel.  It also turns out that
x11vnc can be invoked from a ssh session as well, and by default will only run
for a single connection instance.  And finally, it turns out that
xtightvncviewer lets you customize the ssh command you use to create the "-via"
tunnel, by simply setting an environment variable.  Schweet.

So, here's my (extremely simple) script for accessing the desktop on any machine
you have ssh access to:

<pre>
    VNC_HOST=`echo $1 | awk -F: '{print $1}'`
    DISP=`echo $1 | awk -F: '{print $2}'`
    if [ "x$DISP" = "x" ]; then DISP=0; fi
      
    VNC_VIA_CMD="ssh -f -L %L:%H:%R %G x11vnc -localhost -rfbport 5900 -display :$DISP; sleep 5" 
    export VNC_VIA_CMD
      
    exec xtightvncviewer -via $VNC_HOST localhost:0
</pre>

Again, this script depends on the "x11vnc" and "xtightvncviewer" packages, and
is trivial to extend to work through HP's firewall, as long as you can get a ssh
session through the firewall.  It invokes x11vnc with the "-localhost" option,
so the only way to connect to the session is by first logging into the machine
via ssh or some other tunneling method.  So it's quite secure.
