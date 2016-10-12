---
date: 2013-10-24T00:00:00Z
description: Setting up OS X to mount a share on Login and Wakeup to a fixed location
tags:
- mac
- osx
- mount
- shares
- scripting
title: Mount shares on OS X on Login and Wakeup
url: /2013/10/24/mount-shares-on-login-wakeup/
---

I bought a macbook air recently. This is my first mac, but I am very familiar with Unix/Linux. This is part of a series of posts on using OS X (Mountain Lion).

[Last time]({{site.url}}/osx-automount-mds-oddity/), I wanted to use automount to mount a Samba share at a fixed location. It didn't work as I expected.

This time around, I am going to go the simple route and use `launchd` to mount the share whenever I login.

> launchd -- System wide and per-user daemon/agent manager

In other words, you can use launchd to launch things. :) It's great!

### Launch a script at login
According to `man launchd`, user specific launching instructions are stored in `~/Library/LaunchAgents`. It's pretty easy to figure out the syntax just by looking at the examples that's already there. But for the comprehensive explanation on the plist format for `launchd`, look at `man launchd.plist`.

I ended up with this:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>com.yellowaxe.berunda-mounts</string>
	<key>ProgramArguments</key>
	<array>
		<string>/Users/kal/usr/bin/berunda-mount-shares</string>
	</array>
	<key>RunAtLoad</key>
	<true/>
</dict>
</plist>
```

It's important to put in the full path to your script instead of `~kal/usr/bin/berunda-mount-shares`.

You can then load this .plist file using `launchctl load ~/Library/LaunchAgents/<plist-file-name>`

#### `berunda-mount-shares`
```
#!/bin/sh

for i in `seq 1 9`; do
  # wait until interface is active again.
  # is there a better way to do this?
  ifconfig en0 | grep -q 'status: active' && break;
  sleep 5
done

ping -q -c1 berunda >/dev/null || exit; # server not up

pw=$(security find-internet-password -w -a kal -s berunda)

mount | grep -q /opt/kyle/berunda/media && exit; # already mounted
mount -t smbfs smb://kal:$pw@berunda/media /opt/kyle/berunda/media
```

The for loop is put in to wait for the network interface to become active. This seems to be needed more for the wakeup step. I added some additional changes for server's status and if the mount is already there.

`security find-internet-password` is a great way to find your password in the OS X keychain. You do have to store the password in the keychain first... The easiest way is to mount the share in Finder and have it remember your password.

*Note that the mount command WILL expose your password on the command line (briefly) since it passes the password to `mount_smbfs`*

### Launching a script when the laptop wakes up

For completeness, I wanted to unmount a share when the computer sleeps and remount it when the computer wakes back up. For this, I turn to: [SleepWatcher](http://www.bernhard-baehr.de/). However, I am not going to install it from the site...

Thanks to a friend, I was turned onto [Homebrew](http://brew.sh/). It's a simple `brew install sleepwatcher` to install this.

Running SleepWatcher involves our old friend `launchd` again. Here is the .plist file to use.

#### `de.bernhard-baehr.sleepwatcher.plist`
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>de.bernhard-baehr.sleepwatcher</string>
	<key>ProgramArguments</key>
	<array>
		<string>/usr/local/sbin/sleepwatcher</string>
		<string>-V</string>
                <string>-s /Users/kal/usr/bin/on-sleep</string>
                <string>-w /Users/kal/usr/bin/on-wakeup</string>
	</array>
	<key>RunAtLoad</key>
	<true/>
	<key>KeepAlive</key>
	<true/>
</dict>
</plist>
```

`on-wakeup` just simply calls `~kal/usr/bin/berunda-mount-shares` and `on-sleep` calls `~kal/usr/bin/berunda-umount-shares`.

And to finish this out.

#### `berunda-umount-shares`
```
#!/bin/sh

ping -q -c1 berunda >/dev/null || exit; # server not up

mount | grep -q /opt/kyle/berunda/media || exit; # already unmounted
umount /opt/kyle/berunda/media
```

So far, this has been working great.

