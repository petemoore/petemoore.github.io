---
layout: post
title:  "ZX Spectrum +4 - kind of"
date:   2020-08-24 14:49:27
categories: spectrum
---

The ZX Spectrum +2A was my first computer, and I really loved it. On it, I
learned to program (badly), and learned how computers worked (or so I thought).
I started writing my [first computer
programs](https://github.com/petemoore/zxspectrum) in [Sinclair
BASIC](https://en.wikipedia.org/wiki/Sinclair_BASIC), and tinkered a little
with Z80 machine code (I didn't have an assembler, but I did have a book that
documented the opcodes and operand encodings for each of the Z80 assembly
mnemonics).

Fast forward 25 years, and I found myself middle aged, having spent my career
thus far as a programmer, but never writing any assembly (let alone machine
code), and having lost touch with the low level computing techniques that
attracted me to programming in the first place.

So I decided to change that, and start a new project. My idea was to adapt the
original Spectrum 128K ROM from Z80 machine code, to 64 bit ARM assembly,
running on the Raspberry Pi 3B (which I happened to own). The idea was not to
create an emulator (there are plenty of those around), but instead, to create a
new "operating system" (probably _monitor program_ would be accurate term) that
had roughly the same feature set as the original Spectrum computers, but
designed to run on modern hardware, at faster speeds, with higher resolution,
more sophisticated sound etc.

What I loved about the original Spectrum, was the ease at which you could learn
to program, and the simplicity of the platform. You did not need to study
computer science for 30 years to understand it. That isn't true with modern
operating systems, they are much more complex. I wanted to create something
simple and intuitive again, that would provide a similar computing experience.
Said another way, I wanted to create something with the sophistication of the
original spectrum, but that would run on modern hardware. Since it was meant to
be an evolution of the Spectrum, I decided to call it the ZX Spectrum +4 (since
the ZX Spectrum +3 was the last Spectrum that was ever made).

Well it is a work-in-progress, and has been a lot of fun to write so far.
Please feel free to get involved with the project, and leave a comment, or open
an issue or pull request against the repository. I think I have a fair bit of
work to do, but it is doable. The original Spectrum ROMs were 16K each, so
there is 32Kb of Z80 machine code and tables to translate, but given that
instructions are variable length (typically 1, 2, or 3 bytes) there are
probably something like 15,000 instructions to translate, which could be a year
or two of hobby time, given my other commitments. Or less, if other people get
involved! :-)

The github repo can be found [here](https://github.com/spectrum4/spectrum4).
