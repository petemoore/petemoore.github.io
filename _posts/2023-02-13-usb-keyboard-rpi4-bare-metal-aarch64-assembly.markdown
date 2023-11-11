---
layout: post
title:  "Bare metal aarch64 assembly USB keyboard driver for Raspberry Pi 400"
date:   2023-02-13 16:08:36
categories: spectrum
---

For my [spectrum4](https://github.com/spectrum4/spectrum4) project, I have
reached the stage where I need to process key presses from the Raspberry Pi 400
keyboard. On the original ZX Spectrum, this was a pretty simple task.  Each of
the 40 keys on the keyboard had a dedicated port/bit combination to read from,
to see if it was pressed. For example to see if the key 'P' is being pressed,
port 0xdfde bit 0 will be clear. If key 'P' is not being pressed, bit 0 will be
set. So to test if key 'P' is pressed, this would be sufficient:

```z80
  ld bc, 0xdfde
  in a, (c)
  bit 0, a
  jr nz, cont
  // code here to handle P being pressed...
cont:
  // continue here
```

Things aren't quite so easy on the Raspberry Pi 400. This post is my way of taking notes as I document the process of reading keystrokes on the Raspberry Pi 400 in bare metal aarch64 assembly.

It starts with initialising the PCIe subsystem. The keyboard is a USB3 keyboard. On the Raspberry Pi 400, the USB3 host controller is implemented on the VL805-Q6, whose firmware provides an xHCI interface, which runs over PCIe. So in order to be able to interpret the key presses, we first need to:
* initialise the PCIe subsystem
* implement an xHCI USB driver to talk to the xHCI interface of the VL805-Q6
* implement a USB keyboard driver on top of the xHCI driver

To make matters slightly more complicated, there are an awful lot of specifications that come into play, some of which (PCI/PCIe) are protected via copyrights and cost several thousand dollars to purchase. Furthermore the VL805-Q6 datasheet is also confidential, its firmware closed source, and the Raspberry Pi firmware hooks that interface with it are also closed source/confidential.

What is on our side though, is that we have Linux, Circle, RISC OS and maybe other open source code that have managed to overcome these obstacles, so they will be our primary resource when trying to untangle all of the required steps.

The key files for our reverse engineering are the following ones:

## Linux


