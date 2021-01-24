---
layout: post
title: Welcome to my github blog!
---

Alright, looks like I'm all set up! Welcome to my new blog! I'll update this weekly at a minimum focussing on the progress of my various projects including PJIT and the Buffy Acellerator. What is PJIT and Buffy you may ask? Glad you asked.

![The Buffer Acellerator](https://github.com/nonarkitten/nonarkitten.github.io/blob/master/images/Buffy.png?raw=true)

**The Buffy Acellerator**, so named because of it's Vampire killing abiilty, is a zero-FPGA, zero-CPLD, pure-CPU accelerator for the Amiga Computer. It is designed to fit inline with a 64 DIP socket and should be compatible with the Amiga 500, 1000, 2000 and CDTV. It's based on the amazing Octavo OSD335x System-in-Package which hosts a 1GHz ARM Cortex-A8 processor (about 2000 MIPS), 512MB or 1GB of DDR3 memory as well as various peripherals. It runs PJIT from a small eXecute-in-Place (XiP) Flash ROM and is not designed to run and general purpose operating system itself.

The big reason I chose this processor is because it has the General Purpose Memory Controller (GPMC) which allows it to **DIRECTLY** interface with any asynchronous bus (like the 68000's), and has dual realtime controllers which can offload the various additional bus control signals used by the 68000. This permits the CPU to focus on instruction execution alone.

The work-in-progress design of the board may be found over at [Open Source Hardware Lab](https://oshwlab.com/Renee/buffy2).

**PJIT** is the runtime for the Buffy Acellerator and provides 68000 emulation. It is a threaded JIT which can approach 1000 MIPS of performance or roughly equivalent to a 1200MHz 68040 (hence the PR1200 moniker on the Buffy). However, to play it safe (since not all features are implemented yet), I'm only promising 400MHz performance or better.

PJIT is entirely deterministic and runs real-time. This is in stark contrast to normal JIT engines which can cause jitter when large blocks of code have to be precompiled. It's threaded model does mean that it's not necessarily as fast as a perfect-JIT, but I think the overall performance and user experience will be much greater. Also, because PJIT is very light on memory, it means most of the 512MB/1GB of SDRAM is available to the guest OS (AmigaOS). A more comprehensive explanation can be found in the [PJIT repository](https://github.com/nonarkitten/pseudo-jit).
