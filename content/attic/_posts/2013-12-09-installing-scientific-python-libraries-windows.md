---
layout: post
title: Installing scientific python libraries in windows
excerpt: "Binaries for installing scientific python libraries"
tags: [python, anaconda]
date: 2013-12-09
comments: true
---

I often have issues installing certain python libraries (such as scipy or opencv) in windows. One solution to this problem is to do it the hard ware and just work your way through the errors as they come, winding your way through a maze of StackOverflow posts and mucking around in the registry. An alternative is to install a scientific python distribution like the [Enthought](https://www.enthought.com/products/epd/) Python Distribution, Continuum Analytics' [Anaconda](https://store.continuum.io/cshop/anaconda/), or [Python(x,y)](http://code.google.com/p/pythonxy/). I've played around with Anaconda and Python(x,y) and they're ok. Anaconda at least is supposed to be a little bit faster than the vanilla python installation, but I haven't done any speed tests or anything.

Tonight I was wrestling with opencv. Anaconda didn't have it in its package manager and I kept encountering roadblocks trying to install it in my generic windows installation (CPython). Luckily, I discovered [this website which lists unofficial binaries for installing a lot of troublesome libraries](http://www.lfd.uci.edu/~gohlke/pythonlibs/) in CPython for windows, both x64 and x32. The page is hosted by UCI professor Chris Gholke. If you see this Chris, you're awesome.

Lifesaver.