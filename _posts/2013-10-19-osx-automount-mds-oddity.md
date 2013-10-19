---
layout: post
title: Automount Oddity in OS X with Spotlight metadata server (mds)
description: "Automount doesn't seem to work well with spotlight's metadata server"
tags: [mac,osx,spotlight,automount]
---

I bought a macbook air recently. This is my first mac, but I am very familiar with Unix/Linux. This is part of a series of posts on using OS X (Mountain Lion).

### A Simple Thing
I have a NAS with a Samba share that I'd like to mount on the mac at a *fixed location* and have it to be usable by my user *always*. The same share might be mounted by another user on the same mac, but it can be at a different mount point. This simplifies things a bit. The mount point doesn't have to have open permission for all users.

### Using Finder
Using Finder, the first attempt is to use `Go -> Connect To Server...`. This automatically puts the mount as `/Volumes/<sharename>`. Not so bad if you have only 1 share to deal with. But if multiple users try to mount the same share, you will end up with `/Volumes/<sharename>-1`, `/Volumes/<sharename>-2`, etc. Not permanent solution.

### Using Automount
Automount in OS X is pretty well documented. See `man auto_master`. On Linux, this is always a nice way to lazily mount a drive on demand when you visit a mapped directory. I thought this would solve my problem.

#### `/etc/auto_master`
```
#
# Automounter master map
#
+auto_master          # Use directory service
/net                  -hosts                -nobrowse,hidefromfinder,nosuid
/home                 auto_home             -nobrowse,hidefromfinder
/Network/Servers      -fstab
/-                    -static
/Volumes/kyle/berunda auto_mount_berunda
```

#### `/etc/auto_mount_berunda`
```
media    -fstype=smbfs    smb://downloader:v3rysekret@berunda:/&
```

Run `sudo automount -vc` and it's good to go.

Now, when I `cd` into `/Volumes/kyle/berunda/media`, there is a slight hesitation and the share is mounted.

```
[~] % mount
/dev/disk0s2 on / (hfs, local, journaled)
devfs on /dev (devfs, local, nobrowse)
map -hosts on /net (autofs, nosuid, automounted, nobrowse)
map auto_home on /home (autofs, automounted, nobrowse)
map auto_mount_berunda on /Volumes/kyle/berunda (autofs, automounted, nobrowse)
//downloader@berunda/media on /Volumes/kyle/berunda/media (smbfs, nodev, nosuid, automounted, nobrowse, mounted by kal)

[~] % ls -ld /Volumes/kyle/berunda/media
drwx------  1 kal  wheel    16K Oct 14 23:23 /Volumes/kyle/berunda/media/
```

This was good until sometimes later. I noticed that I can't access that directory anymore. Here is why:

```
[~] % sudo ls -ld /Volumes/kyle/berunda/media
drwx------  1 root  wheel  16384 Oct 14 23:23 /Volumes/kyle/berunda/media

[~] % mount
...
//downloader@berunda/media on /Volumes/kyle/berunda/media (smbfs, nodev, nosuid, automounted, nobrowse)
```

So whenever the mount point is unmounted -- due to inactivity or manual `umount`, some process owned by `root` is accessing the location and remounting it under `root`. Preventing me from access it. I wonder what process could that?

```
[~] % sudo lsof /Volumes/kyle/berunda/media
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
mds      52 root   16r   DIR  45,10    16384    2 /Volumes/kyle/berunda/media
```

`mds` is the OS X process used to index your drives and to produce the metadata that Spotlight uses to perform searches. My guess was `mds` traversed into the directory and automount, of course, mounted the share.

Then, I tried to add the directory to the privacy section in Spotlight.

![Spotlight privacy]({{site.url}}/media/osx-automount-mds-oddity-spotlight-privacy.png)

This didn't help. Neither did completely disabling `mds` using `mdutil -d /`. I think disabling `mds` using `sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.metadata.mds.plist` would probably work. But I am not willing to give up Spotlight completely yet.

So I need another way. Next time I will look into using `launchd` and [SleepWatcher](http://www.bernhard-baehr.de/)

