---
layout: post
title: Getting PJIT Up and Running
---

Just a small update; we've been working hard at getting the core of PJIT ready to boot up before receiving our first batch of boards. Getting the AM3358 up and running without the benefit of JTAG has been ... interesting. To date we have:
- working clock reconfiguration
- set up of all the caches and mmu
- a timer by which we can measure performance
- our first runs of PJIT with the BogoMIPS test

Initially, our very first run was pretty abysmal, but our BogoMIPS score is now between 400 and 666 using the pedagogical BogoMIPS for the 68030 (which ought to take precisely 4 cycles per loop to complete -- this isn't a compiler or 68K optimization exercise). For comparison, the AM3358 natively gets 1000 BogoMIPS (or 990 in Linux, because Linux sucks), which means our emulator is between 40% and 67% of native performance! Right on target!

I was told it couldn't be done, and for a while I was worried they might be right. But they're not and the PJIT model totally works.

It does depend on aggressive inlining and smart selection of which eight of the sixteen 68K registers we keep in the CPU and which end up in SRAM. But I'll be making that a compile time option so beta testers can play with any combination we like. I'll be going over all the nitty-gritty of this in Part 3 of the Performance Series.

Our next step will be implementing a little debug shell where the user can peek and poke around memory as well as getting the GPMC up and running with what we believe are the correct timings. We should have a code update shortly on the github as well. Stay tuned!
