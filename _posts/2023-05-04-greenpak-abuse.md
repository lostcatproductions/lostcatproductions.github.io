---
layout: post
title: GreenPAK Abuse 101
---

When we originally chose to use the AM3358, one of the major reasons for it was the existence of the GPMC or General Purpose Memory Controller. We incorrectly assumed it would be able to cope with virtually any kind of bus, including the 68000, so we eventually had to add a small CPLD to our board -- the GreenPAK. But it wasn't without its own issues.

Aside from the abysmal number of pins, the snafu regarding the pin numbering reversing between packages and the very difficult to use visual editor (please give us Verilog Renesas!), the big problem we've had with the GreenPAK is it's speed -- it is not a terribly fast chip. While a Xilinx CPLD might have a 5ns pin-to-pin minimum, the GreenPAK has 12ns simply passing through a logic cell and about 20ns simply getting through the GPIO structure!

Shockingly, we're able to deal with most of that, but one thing that we couldn't was the amount of jitter the meagre 25MHz clock created. At this speed a simple flip flop takes 40ns with about a +/-20ns margin of error. This is well outside of the timing requirements for the 68000 and made adjusting the GPMC signals to conform to 68000's almost impossible.

So we overclocked it.

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/clock_coubler.PNG)

Using an old trick from Commodore, we used a delay and an XOR gate to create a 50MHz clock, and guess what? It works.

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/clock_doubled.PNG)

Is this a perfect solution? Well, no, but it does improve the precision of any timing by a factor of two and that's allowed us to hit a record-high of 99.8% bus accuracy! That's unfortunately a long way from 100% which is where we need to be, but wow, we're close.

One of the last remaining hickups was tied to how we're using signals from the GPMC for 68K bus signals, notably we use the Write Enable (WE) signal for both the Read Write (RW) signal and the Upper and Lower Data Strobe (UDS/LDS). This was problematic as we ended up needing to have realtively fixed timing for both these and that wasn't working. We have a new version of the GreenPAK code which now decouples these two allowing us to change both the delay from the WE to the xDS signal start and the extra time for the RW signal at the end. coupled with our new found precision, I'm confident we'll have our timing 100% soon enough.

If we're very lucky, a v0.4 release very soon!

Until then, happy hacking everyone, and May the 4th be with you!
