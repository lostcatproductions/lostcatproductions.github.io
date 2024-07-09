---
layout: post
title: Back to Buffee Full Time!
---

Well, I was laid off. But what sucks for me will be a win for the community as I'm working on Buffee again full-time, and progress has been ... interesting. We're also really happy to see prices falling on some of the critical parts of our BOM, especially the main microprocessor. So how about a little "State of the Union Address?"

|![How low can you go?](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/Buffee_v0.8_render.png)|
| :-: |
|Are we there yet Papa Smurf?|

We can read chip RAM at 100% if we "cheat". Our error rate was as low as 2 errors per 25,600 reads, but that's enough to crash any computer within microseconds of booting. However, there's an old trick I used back in my firmware days of debouncing with three reads -- basically it's `(A&B)|(A&C)|(B&C)`. As long as the bits are right 2 of the 3 times, and we don't see any sort of "clustering" with the read/write errors then we should be good. And so far we are.

Is it fast? Nope. Does it work? Yeah.

Our immediate goal now is to simply show it running at all and we're very, very close. The outstanding issue seems to be with interrupts and weird gremlins I thought might have been because of my CDTV relic. However, it turns out the CDTV is fine (yay!) and it was mu socket tower that had developed some poor connectivity issues and was causing us all sorts of grief. I have dug down to the basement now and we're getting good signals again.

|![How low can you go?](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/Buffee_v0.8.png)|
| :-: |
|Forgive the dust.|

This is also proving out v0.8 as it clearly passes the smoke test (as in, neither the CDTV nor Buffee released the magic smoke). This is very promising ans means that v0.8 may be the final version. Buffee v0.8 includes a number of minor fixes to improve our quality of life, includeing

- bytes are swapped to allow native, big-endian memory operations
- greenpak is properly integrated, not as a mini daughter board
- greenpak 3.3V supply is controllable via the CPU now, forcing a reset if I2C fails
- UART Tx and Rx are corrected to use off-the-shelf TC2030-FTDI cable without mods
- many floating pins tied to ground for better heat and power control
- added external pullups to the I2C lines to improve communication to the greenpak
- fixed pmic power good signal at power on to avoid very rare dead lock
- significantly over-engineered power traces to solve 5V droop

So what's the plan?

We're not worrying about PJIT right now. While that is still the whole point, once we have chip RAM and interrupts running, we're going to run this through an emulator and see if we can get this beast booting. I'll post something here the moment we have anything, though if you want to see me screaming in chat, I'll probably be doing that in the Discord first.

As for the beta boards, yes, those will still work. I'm getting the byte-swapper boards sent back to me and I'll mail them out once I do. They'll come "some assembly required," but everyone will get one.

Hopefully in the next little while, we'll drop a v0.4 that can actually run something.
