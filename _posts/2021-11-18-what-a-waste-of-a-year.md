---
layout: post
title: What a year...
---

I have many words for the last year, none of which I can really express here. Suffice to say that this "chip shortage" has pushed back retail production run into January -- at least at this point -- I believe nothing anyone's saying until I have boards physically in hand. Despite the delays, our beta testing is going along well, but very, very slowly. We're still waiting for the last of the beta boards to come in before we begin shipping, and I absolutely appreciate everyone's patience with this. Software is coming along nicely and we have good timing on the bus and can read and write, so that's progress!

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/buffee_bus_la.JPG)
(Thanks to Stephen for the awesome Logic Analyzer donation!)

Debugging the bus aside, we're well along testing PJIT and the C (slow) emulator. But I wanted to take a small step aside and talk about what we've been developing ON. Say hello to the DIP-64 version of the MiniMig currently under development!

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/minimig_68k.jpg)

While I'd love to say that this is just for Buffee, this opens the MiniMig up to running basically any 68000 accelerator, fast RAM or IDE expansion, adding a ton of utility it never had before. Want a 68030? Ok. Maybe an 8MB+IDE board? Sure. How about a Vampire? Hey, sometimes Buffee gets along with Vampires too. Anyway, this is an amazing bit of kit and of course, can still run the SEC000 at 50MHz giving it near Amiga 3000 speeds.

I'm also taking a stab at fixing up some of the old MiniMig Verilog code to add real Amiga mouse support (quadrature decode) as well as proper scanlines that don't darken the screen along with aspect-correct scaling. Wait, what's that, you haven't heard of this? Well, most scanline implementations just black-out or dim alternating lines which dims the screen. Given that LCD's already have a horrid dynamic range compared to CRTs, it's not great makign the screen dimmer than it has to. The magic is in using what is basically an overlay filter, combined with some horizontal (but no vertical) smoothing. I think the look is fantastic.

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/Workbench1.3_KSL_ARC.gif)

We're also looking at updating some other aspects like removing the antiquated PS/2 ports and supporting the 50MHz mode with Buffee to have a 50MHz bus to chip RAM. Fun times ahead, we just need to be a little more patient.
