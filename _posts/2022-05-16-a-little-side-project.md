---
layout: post
title: A Little Side Project
---

With Buffee sort of languishing in the wind, I took a few hours to slap together another board to ensure my PCB routing skills don't get rusty. Instead of a CPU accelerator or anything like that, this is going back to our original Amiga Replacement Project and prodiving DIP and PLCC sides replacements for all the custom and insanely difficult to get custom chips. First up is Gary, or as we call him, Gary Zander.

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/gary_zander.png)

Gary was a nice toe-dip into the world of CPLD and FPGA. At the moment, this board is tested to be 100% reliable on an Amiga 500 (with the small caveat of not liking fast reboots), but the lack of open-drain and tri-state pins on the Xilinx chip prevents us from releasing this just yet (for all you Amiga 2000 owners). The board pictured above adds some open-drain and tri-state drivers (U2 and U3) as required and we'll be producing and testing these boards soon. And yes, it's fully routed.

Following up on Zander will be Willoe, which is a drop-in Agnus replacement.

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/wiloe_exploded.png)

Again, we already have code and the original OCS schematics, and have learned a fair bit about Amiga timing with Gary. We're currently printing 3D models of the PCB to ensure fit into the PLCC socket before producing any development boards. The rough feature set of Willoe:
- 100 pin TQFP Lattice MachXO2 core (XO2-2000)
- Near instant-on (CPLD-like) functionality
- 16MB of 133MHz, 8-bit pSRAM for AGA+ bandwidth
- 1MB, 2MB Agnus and 2MB Alice compatibility
- Jumpered NTSC/PAL default modes (will also obey pin and software changes)
- Automatic 8372 and 8375 pin detection (universal socket compatibility)
- Many optional features to come

There is news on Buffee as well that we'll post in coming days. Thanks!
