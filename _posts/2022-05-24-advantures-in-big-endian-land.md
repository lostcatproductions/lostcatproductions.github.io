---
layout: post
title: Adventures in Big Endian
---

Wow, what a journey; to start with the Beaglebone finding big-endian worked fairly well to find out its completely broken on Buffee and that big endian is completely unsupported on the AM3358, something the people at TI will repeat in their support forums again, and again, and again. But it's all a lie and big endian does work ... perfectly. So what's up?

A quick aside, there's some confusion about endianess on the ARM. And some may not even get endianess at all to begin with. I'm not going to get into the technical differences of the two -- suffice to say little endian is cheaper in silicon and big endian preserves what I would call "natural data order." ARM is a so-called bi-endian processor meaning that it can switch between little and big endian mode as needed, but there's a caveat: instructions are always executed in little endian mode.

And this is why I **think** people will say the AM335x doesn't supprt big endian. It can't, because the instruction pipeline does not support it. But I also **think** that zero developers are asking for big endian opcodes -- it's just about the data. So TI support staff might not be lying, but they're not being honest either.

Okay, so how's it done?

## Attempt 1 -- Compiler Options

This is seems to be the simplest step. In the General tab of your project properties, change Device Endianess to Big and make sure the compiler is set to GCC. Once you do this, it should compile fine, but when trying to debug, you'll immediately be in a data abort. If you dig a little deeper its because the CPU's in Thumb mode. What?

## Attempt 2 -- Fixing the Compiler

So it would seem that the project settings don't do a lot and we still need to add the -mbig-endian to the list of flags for GCC. You might be tempted to also add the ```--be8``` option to the linker -- don't, and I'll explain that below.

With that done we take another stab at running and *BOOM* linker errors. Why? Because unlike LTS mode, GCC won't ad hoc compile you a big endian run time library. So for now, we're running "bare metal." Running code still blows up becaues now JLINK will complain that we're programming a little endian part with big endian code.

Forehead meet keyboard. Rinse and repeat...

## Attempt 3 -- Target Options

This one's a bit more work. We need to open our connection properties (\*.ccxml) and go into Advanced. Navigate to the CPU (in my case this is CortxA8) and in JLink Scriptfile Path, click browse and create a new file (I called mine jlink.script -- original, eh?) and paste the following into it:

```c
int SetupTarget(void) {
    JLINK_TargetEndianness = JLINK_TARGET_ENDIANNESS_I_LITTLE_D_BIG;
}
```

Super obvious, right? So we're almost there, it programs it starts running again and most likely you're back in the abort handler.

## Attempt 4 -- Removing the be8 Option

Well, when I first went through all this, I included the be8 option to the linker -- naturally. Unfortunately, if you've gotten this far you'll find all the bytes swapped and of course, the ARM core is now executing nonsensical instructions. JLink is doing us the courtesy of swapping all the bytes for us because clearly Segger don't understand endianess on ARM and we're programming a little-endian part with big-endian code ... so we need to leave this option **OUT** to make "bad code" that JLink will "fix" and program our device correct.

This sadly is not as simple as you'd think because the toolset autogenerates the makefile based on the project options. So, you need to strip that all out and use a manually created makefile. While it might be an option to start with a plain makefile project, I found this to be no less painful to get up and running -- actually, once you know what to turn off, starting with a regular CCS project first is a lot easier.

And in the end, the love you take is equal to the love ... you make.

So now big endian is working. It works for DRAM. It works with the GPMC. The only exception is that the registers need byte swapping if you're reading/writing in 16- or 32-bit because those remain in little-endian mode. And of course, there's the matter of making a GCC big-endian C standard library and IO programming, but knowing that this actually works means proceeding.

## How This Relates to Buffee

When we first encounted this problem, we decided that the best option for now was to physically swap the bytes on the GPMC interface. This means word reads and writes have a built-in byte swap and since this is most operations (e.g., opcode fetches), this seemed okay. For byte read/write we xor the address and for long read/write we swap the high/low portions of the long word. This leads to one extra opcode in all 8-bit memory operations and one (read/write) or two (read-modify-write) extra opcodes on 32-bit memory operations.

This would hurt performance a fair bit -- not when executing through the GPMC (which operates at a couple orders of magnitude slower than the CPU) -- but when running in 32-bit mode from the internal DRAM. We cannot hardware byte-swap words there and we'd have to add extra conditionals to see if we're running through the GPMC or DRAM, perform the required swapping. Memory overheads will quickly baloon out of control and our goal of 1000 MIPS would have been likely unachievable.

## The Buffee Swapper

So ... you have an Alpha or Beta board and want to know what this means for you? Well, we're making the Buffee Swapper that will swap the external data bus bytes. This board will be a small shim that sits between Buffee and the motherboard and allows Buffee to run in proper big-endian mode. I will be shipping this out **FREE** to every Alpha/Beta user.

Future Buffee boards will have this fixed and will not need the shim in place.

## What about 24-bit Addressing

Well, that's still a pain and needs a ```bic``` instruction for nearly every memory operation to clear out the high 8-bits. I say nearly, because the absolute addresssing modes can be masked by PJIT before hand. But every other addressing mode using an address register? Nope, those need to be masked out every time ... ugh. If anyone has ideas on this, feel free to post on our now-public github or drop by on Discord and let me know.
