---
layout: post
title: The "Amiganess" of Buffee
---

There seems to be a bit of confusion with Buffee and perhaps that's my fault. My father's favourite saying is, "a word to the wise should suffice" and to me, that always meant be terse and to the point and people should figure it out. I've mastered the art of being terse to the point of sometimes not saying much at all -- so I'll say it here, loud and clear -- the Buffee supports everything your Amiga can do right now.

- Autoconfig and Zorro-II
- In-socket expansions
- All 2MB of chip RAM
- Any fast RAM you might have
- Any "ranger" RAM you might have
- The original OCS all the way to ECS
- Floppy drives
- Mice
- Joysticks
- Custom ROMs
- CDTV expansion ROMs
...

If your existing 68000 can do it, then Buffee can do it. Just some 2000 times faster. But I did say that Buffee is not "Amiga specific" and that we cannot support things like Autoconfig, didn't I? Well, I meant that the memory and peripherals **INTERNAL TO BUFFEE** cannot be automatically enabled and used by AmigaOS. Or EmuTOS. Or Mac System. Because they all do this differently and they all expect memory and peripherals in different locations.

What this means is that from a ***default*** state, Buffee will operate in "ultra compatible mode" and will require that the user or retailer preconfigure Buffee before hand. This includes such things as

- Enable CPU local RAM in 32-bit memory (disabled by default)
- Set the location and amount of this RAM
- Enable data and/or instruction caches for any memory block (by default only the instruction cache is enabled in 24-bit memory, all caches are enabled for 32-bit memory)
- Enable PJIT instruction cache for any memory block (by default, PJIT ICache is enabled only on 32-bit memory)
- Set the CPU base clock PLLs (presets for 275 to 1000 MHz)
- Set the CPU clock divider (from 1 to 1/256th of the base clock)
- Select between 68000 or 68030 instruction set (68000 default)
- Select between No FPU, 68882, 68040 or fast 68040 FPU modes (No FPU is default)
- Enable 68K MMU emulation
- Preload and remap up to a 2MB ROM image (with or without 68K MMU enabled)

None of these settings need to be set by a CLI utility on each boot (though they can be); these can be saved into the EEPROM memory which will be retained while powered off for decades! 

What this all means is that when you first run Buffee (especially if you're a beta tester), Buffee will not perform at full speed. Not even close. But once it's configured, it will remain in that configuration and will boot every time with full RAM and full speed. This also means that you can make temporary changes (e.g., the speed) to run games or other software titles that might not run well at 1000 MIPS.

So Buffee would then be ***VERY*** Amiga-specific.

I hope this puts some concerns to rest, and if you need to pick my brain further, please drop by our Discord server.
