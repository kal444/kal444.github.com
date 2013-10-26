---
layout: post
title: RVM wrangling on OS X
description: "RVM wrangling on OS X"
tags: [rvm,xcode,apple,osx,gcc]
---

As part of setting up Jekyll to run this blog, I needed to install several gems. I already have [RVM](https://rvm.io/) installed.

While installing RedCloth, I get this:

```
Installing RedCloth (4.2.9)
Gem::Installer::ExtensionBuildError: ERROR: Failed to build gem native extension.

    /Users/kal/.rvm/rubies/ruby-2.0.0-p247/bin/ruby extconf.rb
checking for main() in -lc... *** extconf.rb failed ***
Could not create Makefile due to some reason, probably lack of necessary
libraries and/or headers.  Check the mkmf.log file for more details.  You may
need configuration options.
...
/Users/kal/.rvm/rubies/ruby-2.0.0-p247/lib/ruby/2.0.0/mkmf.rb:434:in `try_do': The compiler failed to generate an executable file. (RuntimeError)
You have to install development tools first.
...
Gem files will remain installed in /Users/kal/.rvm/gems/ruby-2.0.0-p247/gems/RedCloth-4.2.9 for inspection.
Results logged to /Users/kal/.rvm/gems/ruby-2.0.0-p247/gems/RedCloth-4.2.9/ext/redcloth_scan/gem_make.out
```

Hmm, I thought I already installed development tools (Xcode on OS X).

I typed `gcc`. "command not found". Odd.

Oh, look at this, Xcode was just updated and I don't have the command line tools

In Xcode 5, go to `Preferences... -> Downloads` and try download the command line tools. Hah, I need to sign up with an Apple developer account. Here is the [link](https://developer.apple.com/register/) to register as an Apple developer for free.

Finally, command line tools are installed.

Now,

```
rvm get head 
rvm reinstall 2.0.0
```

Now, install the gems again. Success!

P.S. During this episode, I learned about [rbenv](https://github.com/sstephenson/rbenv) via some google battles. Looks really good, and I am going to switch to it instead of RVM.
