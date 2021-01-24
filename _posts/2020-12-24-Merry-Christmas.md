---
layout: post
title: Merry Christmas!
---

Well, 2020 has just flown by. I'm sure I'm not alone in saying good riddance. It's the year that broke the world and I sure hope that when we start rebuilding, that we don't make all the same mistakes.

We had a very busy year over at CoolIT and I'm super proud to have wrapped up a marathon of debugging the firmware for the CHx80 which is now the most stable and feature rich version we've ever released (and due to code sharing will be the new base for all our other units). It seriously has chewed into my desire to do much coding on the side though, but I also kicked off this blog and am committed to seeing Buffy and PJIT through!

I have three major side projects on the go in 2021:
- _**PJIT**_, the pseudo-JIT designed to be nearly as fast as traditional JIT while being much quicker
- _**Buffy**_, an in-line DIP 68000 accellerator based on the Octavo OSD335x board offering 1BIPS of 68000 power
- _**Pi400 Super Expander**_, an Amiga in a cartridge for the Pi400 running its own CPU emulation

For 2021 I plan on having the first beta version of PJIT that can run on a Beaglebone Black with the BeagleWire sometime around February. The [BeagleWire](https://www.crowdsupply.com/qwerty-embedded-design/beaglewire) is a small FPGA and SDRAM board which we can use to recreate the Amiga chipset sans CPU to test against without hitting a real Amiga. I only have a few and destroying them prematurely doesn't do anyone any good!

And then wrap up the initial routing of the main board sometime around March. We'll be getting the design professionally reviewed by Octavo themselves before printing the first run and then assembled using some local supplier. Of course, it is also 100% open source, but may be difficult to assemble. Hopefully I have a complete Buffy up and running before summer, but I expect 2-3 respins before it's ready for consumption.

This will lead later in 2021 to the Pi400 Super Expander which uses an similar FPGA and SDRAM to implement the full Amiga chipset, offloading that work for Amiga or 68000 emulators (such as Emu68, PJIT or Cyclone 68000). You could think of this as the opposite of the PiStorm. It will be 100% open source and the design may be fully assembled through [JLCPCB](https://jlcpcb.com) by anyone, no (or minimal) soldering required!

Well, Merry Christmas everyone and Happy New Year!
