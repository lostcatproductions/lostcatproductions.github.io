---
layout: post
title: Bootloader Progress
---

Following a catastrophic loss of the bootloader work to my sheer negligence, I've gotten it back up to where I last left it. The bootloader is responsible for initializing the hardware and it has not been easy getting this far, but at the moment, everything is working except the 68K bus. That is, of course, the most important thing Buffee needs to do, but it sure wasn't everything the bootloader needs to do.

What it does:
- initializes the core and peripheral clocks, caches, DDR and MMU
- initializes the GPMC, GPIO, SPI, I2C and UART peripherals
- tests and reprograms the GreenPAK; this will simplify production as well since it's now just programming one device
- tests and loads the EEPROM settings or restores defaults on uninitialized or bad EEPROM
- tests and sets the PMIC for maximum performance (1GHz)

Presently, this takes about 77ms. Aside from loading and running PJIT, the only buggy bit is the GPMC and GreenPAK. We're able to read ROM fine, poke around Agnus and Paula, but the CIA access is still broken and the output from my Logic Analyzer strongly suggests the logic on the GreenPAK is wrong as well.
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
...
[GPMC] Read words (LE:BE)
[GMPC] $00000000: 1111:1111  f94e:4ef9  fc00:00fc  d200:00d2 
[GMPC] $00000008: 0000:0000  ffff:ffff  2200:0022  0500:0005 
[GMPC] $00000010: 2200:0022  0200:0002  ffff:ffff  ffff:ffff 
[GMPC] $00000018: 7865:6578  6365:6563  3320:2033  2e34:342e 
[GPMC] INTENAR: 0000 (should be 0)
[GPMC] JOYxDAT Check: 0xf0f0 0xf0f0 (should be close to 0xF0F0)
[GPMC] JOYxDAT Check: 0xa6a7 0xa4a4 (should be close to 0xA5A5)
[GPMC] Performing benchmark with CIA (~1.5s)
[GPMC] Read 131072 words in 0.16 s (0.74 MB/s)
[GPMC] CIAA timer A Start: 0x00000000
[GPMC] CIAA timer A End: 0x00000000
[GPMC] CIAA timer A Elapsed: 0
```

Also, check out the awesome ASCII art!

So why are the CIAs and GPMC not working?

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/Thinking.png)

Well, right now, I'm not even getting the VPA back from Gary -- that signal seems stuck high with nowhere to go. There is a remote possibility the Amiga I'm using to test with is broken. I sure hope not, these are getting expensive to replace.

With the GPMC, the data strobes are somehow getting set BEFORE Address Strobe. Right now, I don't know if this is a timing issue in the GPMC configuration or the GreenPAK. I'll have to go through each, again.
