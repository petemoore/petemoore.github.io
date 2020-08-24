---
layout: post
title:  "Open Source Alternative to ntrights.exe"
date:   2020-08-24 14:32:13
categories: general
---

If you wish to modify LSA policy programmatically on Windows, the
`ntrights.exe` utility from the Windows 2000 Server Resource Kit may help you.
But if you need to ship a product that uses it, you may wish to consider an
open source tool to avoid any licensing issues.

Needing to do a similar thing myself, I've written the `ntr` open source
utility for this purpose. It contains both a standalone executable (like
`ntrights.exe`) and a Go library interface.

The project is on github [here](https://github.com/petemoore/ntr).

I hope you find it useful!
