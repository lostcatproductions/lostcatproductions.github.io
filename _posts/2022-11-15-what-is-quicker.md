---
layout: post
title: Performance Series Part 3 -- What Does Quicker Even Mean?
---

In [Part 1](https://www.buffee.ca/why-is-pjit-fast/) we looked at how PJIT can be faster than existing traditional interpreters and in [Part 2](https://www.buffee.ca/why-is-pjit-slow/) how it might be slower than existing traditional recompilers such as Emu68 and UAE. While PJIT appears to live in a happy middle ground, there is one compelling reason why PJIT is better than either strategy -- because PJIT is quicker than either of these methods. But what does that even mean?

## The Flaw of JIT

So you boot up your Amiga JIT and run SysInfo to find the ancient user interface doesn't even have enough room to print all the digits because you're literally thousands of times faster than an original Amiga. What's missing here, however, is the overhead in the emulation itself that the benchmark is not -- and cannot -- measure.

Let's take a more absurd example. Suppose you start a Lightwave render and pause the emulator for a week in the middle. In real life, the render took a week to run but to the emulator it was only a few minutes. JIT, by design, introduces thousands of "micro-pauses" or what I call 'jitter' in the program as the emulation needs to stop while code is recompiled by the JIT, and no benchmark software will ever expose these.

When you run SysInfo, something like this happens.

1. Run SysInfo an click Speed
2. JIT recompiles SysInfo benchmark
3. SysInfo performs benchmark

This is really over simplified, but see the problem? Step 2 is not counted in the **TIME** measured in Step 3. So SysInfo sees that it manages say a million loops in one second when actually, two seconds have passed in real time. SysInfo cannot know of this time because the emulation wasn't running during this time. 

When Step 2 is included, the performance of JIT drops anywhere from 20% to 99.5%. So JIT can appear fast, but it is not quick. **_Quick_** is how long it takes to get from **READING** the first real 68K opcode to the point where we actually **EXECUTE** it. With traditional JIT this can be no time at all on sequential instructions, to thousands of cycles on the very first instruction of a new code block.

### Pseudo JIT

Pseudo JIT takes a very different approach and the core of this is in how PJIT manages the cache. In traditional JIT, the translated code blocks are stored in a heap, linked with a hash map. In PJIT, the cache is structured like a physical processor cache, comprised of sets and lines where each ARM instruction corresponds to one 68K instruction, no more and no less.

A set is normally about 4KB in size. This memory is cleared with a branch to a subroutine that determines the 68K program counter, fetches the opcode and its corresponding subroutine and then replaces the original branch in the cache with a new branch to the opcode handler before finally calling it. Violations with the instruction cache are handled by simply keeping the entire JIT cache out of the CPU's instruction cache but still in the L2 cache. The entire self-replacement routine takes as little as 21 cycles (compiled with -Os).

The average number of instruction per 68K opcode is currently about 3 cycles, meaning that this has an 87.5% overhead and on average can execute about one 68K instruction in 24ns, or a little faster than the speed of a 40MHz 68040 -- on par with a "good interpreter". This overhead is only incurred on the first pass of the instruction, and every subsequent execution will only need 3 cycles -- on par with a "good JIT". So what we've managed here is the quickness and predictability of an interpreter with the potential peak performance of the best JITs.

## Some Updates

Since Part 1 and 2, we've found a few other areas of general improvement. 

### Fixed Registers

```
  bl #offs                 ;@ to get here...            

;@ ---------- [1000] move.b d1, d0 uses Op1001 ----------
Op1001:

  ldrb r2,[r7,#4]          ;@ load byte from D1
  strb r2,[r7]             ;@ store in D0

  bx lr                    ;@ and return
```
At the end of Part 1 and 2, we left off with this snippet of code for MOVE.B. It was slightly wrong: it's missing the CMP and would not set the N flag correctly.

Now, unfortunately, 32-bit ARM does not have enough CPU registers to store ALL of the 68000's registers. We performed a lot of code profiling and found D0 to D3, A0 to A3 and A7 to be the most used registers (by a LOT!), and playing with various code-generation options, found that address registers are much more important to keep in the CPU than data registers. So now, registers D0, D1 and all the address registers are now preserved in the CPU. This has quite the impact on the generated code.

With byte and word operations, we're not better off:
```
  bl #offs                 ;@ to get here...            

;@ MOVE.B D1,D0
opcode_1001:
	sxtb    r0, r4           ;@ move and sign-extend D1 into temp
	cmp     r0, #0           ;@ set our flags
	bfi     r3, r0, $0, #8   ;@ insert the low 8-bits into D0
  
	bx      lr               ;@ and return
```

However, with long operations using only D0, D1 or any address registers, it's quite a bit better.
```
  bl #offs                 ;@ to get here...            

;@ MOVE.L D0,D1
opcode_2200:
	movs    r4, r3           ;@ move D1 to D1 and set flags
  
	bx      lr               ;@ and return
```

Because this is just one ARM instruction (not counting the return), it is now elligible for inlining. Mixing local and memory based registers is somewhere between the two, usually being two ARM instructions. There's still a lot of room for improvement here, but I'm putting a pin in any more microoptimization until we get basic bus operations working solid and have a way of even running PJIT from Visual Studio Code.

### Extension Words

Oh the humanity! So extension words are now handled by the parser and emit ARM opcodes. For example, to load a 16-bit value, we'd use MOV rX, #immd. For 32-bit values, we'd pair that with a MOVT rX, #immd. This works out remarkably well. For the more complex addressing modes, we branch to special handlers which return a simple 32-bit value for the instruction to use. For 68020 addressing modes, this will become more complex, but the same basic pattern will still hold -- all the extension fields simply get handled BEFORE the actual opcode.

This has the side effect of "merging" several addressing modes. The address register indexed and displacement modes are now handled the same, and both absolute and PC-relative addressing modes are all handled the same. This has shrunk the opcode table considerably.

This also simplifies the whole return-to-PJIT-cache logic in that we don't need to skip anything for "data."

### Division and Other Hellish Opcodes

Would you believe our CPU doesn't have hardware division? Me neither at the time. Like Division, there are a few opcodes that need a lot more effort and this really skews the "3 cycles" I mention above. In fact, more than 25% of all the opcode handlers are fewer than two cycles, but there are a few obscenely long ones that messes everything up, including MOVEP, MOVEM, DIVU/DIVS, ROXL/ROXR, ABCD/SBCD/NBCD.

Each of these have to drop into more traditional "interpreter" handlers right now. I have been refactoring these a little to sort-of-inline a more complex handler for the common stuff with per-opcode offload. This is messy though and doesn't pass the smell test. There has to be a better way.

### Still To-Do

Well, there are only two real hangups left. At the moment, getting the board programmed has been the biggest headache and we've toasted a few beta boards already (thankfully, we have not toasted any of our test Amigas). Kipper's offered to try resoldering the bricked chips -- I wish him the best of luck. We're still trying to nail down timing with the CIA chips (e.g., speaking 6800), but at the moment ROM and the custom registers are working tickity-boo. I can't test chip RAM until I can do that, since ROM is blocking access. We have this all running through the Buffee Bootloader and here's a dump from the most recent run.

```
              ____ ______________
  ______     |    |   \__    ___/
  \  __ \    |    |   | |    |
  |  |_) )\__|    |   | |    |
  |   __/\________|___| |____|
  |__|

[BOOT] Build Date Oct 21 2022, Time 17:49:02
[BOOT] Image 402F0400 ~ 402F600C (23564 bytes)
[I2C0] Scanning bus...
[I2C0] 000_10xx ($8~$A) GreenPAK Detected.
[I2C0] 010_0100 ($24) PMIC Detected, Nitro mode enabled.
[I2C0] 101_0000 ($50) EEPROM Detected.
[BOOT] Completed in 0.07710 seconds.
MENU
1. Quick-test DDR memory.
2. Dump first 4K of SPI Flash.
3. Quick-test SPI flash (Warning: destructive).
4. Erase whole SPI flash.
5. Program SPI flash with flash loader.
6. XMODEM-1K upload image to DDR.
7. Program SPI flash with image.
8. Test GPMC.
9. Test printf.
A. Run Native BogoMIPS test.
G. Scan and verify GreenPAK.
I. Scan I2C Bus.
P. Program GreenPAK.
S. Set GreenPAK Address.
X. Reboot.
Ready.
] 1
[DDR0] Read 01010101 from 80000000, expected  01010101
[DDR0] Read 02020202 from 80000004, expected  02020202
[DDR0] Read 04040404 from 80000008, expected  04040404
[DDR0] Read 08080808 from 8000000C, expected  08080808
[DDR0] Read 10101010 from 80000010, expected  10101010
[DDR0] Read 20202020 from 80000014, expected  20202020
[DDR0] Read 40404040 from 80000018, expected  40404040
[DDR0] Read 80808080 from 8000001C, expected  80808080
Ready.
] 3
Are you sure [y/N]? y
[FLASH] Flash Device ID: 1f16
[FLASH] Erasing page 0
[FLASH] Writing pattern block
[FLASH] Read: AA55AA55 AA55AA55 AA55AA55 AA55AA55
[FLASH] Writing zero block
[FLASH] Read: 00000000 00000000 00000000 00000000
[FLASH] Tests passed
Ready.
] 8
[GPMC] ROM Check: exec 34.2 (28 Oct 1987)
[GPMC] Bytes NOT swapped
[GPMC] Raw bytes:
[GMPC] $00000000: 11 11 4e f9 00 fc 00 d2
[GMPC] $00000008: 00 00 ff ff 00 22 00 05
[GMPC] $00000010: 00 22 00 02 ff ff ff ff
[GMPC] $00000018: 65 78 65 63 20 33 34 2e
[GPMC] Read words (LE:BE)
[GMPC] $00000000: 1111:1111 f94e:4ef9 fc00:00fc d200:00d2
[GMPC] $00000008: 0000:0000 ffff:ffff 2200:0022 0500:0005
[GMPC] $00000010: 2200:0022 0200:0002 ffff:ffff ffff:ffff
[GMPC] $00000018: 7865:6578 6365:6563 3320:2033 2e34:342e
[GPMC] Read longs (LE:BE)
[GMPC] $00000000: f94e1111:11114ef9 d200fc00:00fc00d2
[GMPC] $00000008: ffff0000:0000ffff 05002200:00220005
[GMPC] $00000010: 02002200:00220002 ffffffff:ffffffff
[GMPC] $00000018: 63657865:65786563 2e343320:2033342e
[GPMC] Performing benchmark with ROM (~1.5s)
[GPMC] Read 1048576 words in 1.35 s (1.47 MB/s)
[GPMC] JOYxDAT Check: 0xf0f0 0xf2f3 (should be close to 0xF0F0)
[GPMC] JOYxDAT Check: 0xf2f3 0xffff (should be close to 0xA5A5)
[GPMC] CIAA timer A Start: 0x00ffffff
[GPMC] CIAA timer A End: 0x00ffffff
[GPMC] CIAA timer A Elapsed: 0
Ready.
] A
[TEST] Performing Native BogoMIPS benchmark, warming up
[TEST] Warm up complete, starting pass 1
[TEST] Native BogoMIPS 999.99
[TEST] Loops: 536870912, duration 1073 ms
Ready.
] X
Are you sure [y/N]? yCCCCCCCCCC
```

If you are brave. And I do mean incredibly brave, you're welcome to download and play with the bootloader. You can upload this via UART (using XMODEM transfer) or JTAG and the appropriate cables. I do not recommend anyone muck with the GreenPAK programming yet until we're 100% sure this won't brick anymore Buffee's. These CPU's are unobtanium as it is, and I cannot replace any broken boards. You've been warned.

[buffee_bootloader.bin.zip](https://github.com/lostcatproductions/lostcatproductions.github.io/files/10016799/buffee_bootloader.bin.zip)

Eventually, we'll omit the console (that's most of the 77ms of boot time) unless someone hits a key. That way the bootloader can be used by JTAG to initialize the hardware and bootstrap PJIT through JTAG. Of course, we'll also have to reenable the XMODEM upload which will also allow full programming (including PJIT) via UART, if you choose. Right now, that option won't work. I will have an update shortly though which does.

On a final note, appologies for no pics in this one. I know, it's a little dry. I should have something a little more interesting next time, I promise.
