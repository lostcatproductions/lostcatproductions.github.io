---
layout: post
title: Beta Buffees Ordered!
---

With a ton of testing behind us, we've pushed on to the second round of Buffee boards and have officially submitted the order for sixty beta Buffees. While we had hoped that the alpha version would just work, we were not so lucky and Buffee v0.5 has a new little friend -- a GreenPAK SPLD (Simple Programmable Logic Device) and we've ensured that on this version every signal that is not bound to the SPLD, is bound to the PRU (Programmable Realtime Unit) which is almost as good as the SPLD, just not quite as fast. This doesn't mean the alpha boards are useless though!

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/Buffee_v0.5.png)

The little SPLD is U3, cozy among all the passive parts to the left of the main processor. The GreenPAK can be fully programmed from the CPU, no additional steps are required to use and/or program it, and it should be sufficiently complex to properly arbitrate the oddball 680x0 bus. All the other "slow" signals, like bus arbitration and the 6800 synchronous signals all go to the PRU, which, again, can be easily reprogrammed by the main CPU and no special steps need to be taken.

Other than that and some artwork clean up, there's not a ton of differencee with v0.5. Oh no, wait, yeah, there's a few.
- Flash SPI data lines were swapped (this is possible to fix on the alpha)
- Made Flash SPI lines length matched to ensure good data up to 104MHz
- Power rails changed to support down to 2.7V (alpha will brown out at 4.5V)
- JTAG had no system reset only target reset (makes debugging annoying)
- CLK pin may work either as an input or output (improved TF1200 support)
- A few extra passives that weren't needed were removed
- Switched to lower-profile headers to make Buffee the same height as a 68000
- And added the GreenPAK for some simple programmable fun
- Fixed the silkscreen to align with the BGA orientation

Oof. So you have an alpha and you're thinking right now that you're holding some junk -- well, yeah, but there's hope! In fact, the Buffee Project is offering you four options.
1. You can fix the data lines and the alpha will still run in the TF1200.
2. We can send you a new PCB and you can try and swap it yourself or source the CPU.
3. You can send it back and we'll try and replace the PCB for you.
4. We can send you a TSSOP GreenPAK and you can try and bodge the alpha yourself.

We're seriously recommending the first. Like seriously. Yes, it's possible to remove and replace a BGA chip, but on the SAME chip, it's likely to result in catastrophic damage to the inner components of the CPU. And while the GreenPAK bodge is in theory possible, it would be fragile and very difficult to do correctly as at least one of the pads from the BGA is required and not brought outside of it's footprint.

So beta's are off and with all the chip-shortages, we're actually just barely coming in under the MSRP with almost no margin left. Also, we're stuck with a six week build time as there are some lead times we have to absorb. Such is life and I appologise, but there's really nothing we can do about this right now.

Finally, some have commented that the PJIT github is "incomplete" (that's probably the nicest word people have used). We're keeping the core emulation of PJIT internal until retail launch, just to stop cloners from beating us to the finish line before us. Beta binaries will be provided to the testers through our Discord server. I cannot believe it's gotten this bad in the retro community -- if we want new hardware and new software, we have to support eachother and not undermine one another.

And sorry for the long wait. I should have Part 3 of the Performance Series up next week.
