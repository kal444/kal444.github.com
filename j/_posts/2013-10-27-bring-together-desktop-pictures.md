---
layout: post
title: Combine OS X default wallpapers with customized ones
description: "Combine OS X default wallpapers with customized ones"
tags: [apple,osx,wallpaper,configuration]
---

I just upgraded to OS X Mavericks, but this applied equally to OS X Mountain Lion.

As it turns out, in addition the default desktop wallpapers included by Apple, there are a set of images to be used by the screen saver. These are great as desktop wallpapers too!

You can find where there are here: [Secret Wallpapers](http://apple.stackexchange.com/a/106039)

But, that's not what this post is about.

Let's say you want to use those pictures AND you want to use the default ones in a rotational setting. Since you can't move the desktop pictures folder from Apple to Folders, you can only pick either you own folders or the Apple one.
![Desktop Pictures]({{site.url}}/media/desktop-pictures.jpg)

I don't want to make a copy of the provided desktop pictures. First I tried to create a symbolic link to the Desktop Pictures directory, but Desktop & Screen Saver dialog is smart enough to know that it's the same folder as the one under Apple.

This works though:

```
mkdir -p ~/Pictures/Apple\ Provided
cd ~/Pictures/Apple\ Provided
for i in /Library/Desktop\ Pictures/*.jpg; do ln -s "$i" "`basename $i`" ; done
```

Then, add `~/Pictures/Apple\ Provided` to Folders and select Folders.

That's it.
