---
layout: post
title: Buffee Firmware Beta v0.3
---

Progress on "PJIT 2.0" has gone well so far and we're now into our third beta release. Executing code is still problematic when the bus timing is still not 100%, but we're getting there. Users are welcome to download the binary and play around, we'd be absolutely thrilled if anyone could play around with the GPMC settings and let us know what works and what doesn't.

# Programming

You're welcome to program this either with a 3.3V UART or JTAG connection using either the [binary here](https://github.com/nonarkitten/pseudo-jit/releases/download/v0.3/pjit.bin) or the [ELF file here](https://github.com/nonarkitten/pseudo-jit/releases/download/v0.3/pjit.elf) respectively. When using UART, set your terminal program to 115,200 baud and 8N1 parity. When you first connect, you should see a steady stream of 'C' characters at which point you can XMODEM transfer the binary into RAM. It should start immediately.

## Flashing the Firmware

Among the various options is to program the firmware to SPI Flash. I don't recommend doing this yet.

## Recovering the Flash

If you didn't listen and anything bad happens, you should be able to recover by rebooting the system while holding the Escape key in the terminal window. You should see a message saying that flash has been erased and you'll be back to the repeating 'C'.

# Firmware

Upon boot, you should see the PJIT banner as well as some boot up noise. The firmware includes both the bootloader and the 68K emulator so there is no second-stage required.
```
              ____ ______________
  ______     |    |   \__    ___/
  \  __ \    |    |   | |    |
  |  |_) )\__|    |   | |    |
  |   __/\________|___| |____|
  |__|

[BOOT] Build Version, v0.3, Date Apr 18 2023, Time 14:00:26
[I2C0] Scanning bus..
[I2C0] EEPROM Detected ($50)
[I2C0] Settings loaded, last boot was good
[I2C0] GreenPAK Detected ($08~$0B) 
[I2C0] GreenPAK Protection Bits: $00 $00 $00
[I2C0] GreenPAK good
[I2C0] PMIC ID: TPS65217C
[I2C0] PMIC DCDC voltage: 1.35
[I2C0] Nitro mode enabled
[CLK7] Main bus clock measured at 7.158
[GPMC] Trimming Core PLL to: 966
[GPMC] SYNC VIOLATION: t->CSOFFTIME >= (t->ACCESSTIME+1)
[BOOT] Initializing cache
[BOOT] Initializing opcode tables
[BOOT] Image 402F0400 ~ 40302560 (74080 bytes)
[BOOT] Stack 403081E5 ~ 4030FFF8 (32275 bytes)
[I2C0] Saving and verifying settings (CRC=823A)
[BOOT] Completed in 0.19795 seconds

MENU
----
TESTS:
 1. Quick-test DDR memory
 2. Dump first 4K of SPI Flash
 3. Quick-test SPI flash
 4. Test GPMC
 5. Test printf
 6. Run Native BogoMIPS test
 7. Run PJIT BogoMIPS test
SETUP:
 J. Jump to PJIT
 C. Set E Clock Divider
 R. Run MCL68K
 S. Manage SPI Flash
 E. Manage EEPROM Config
 G. Manage GreenPAK
 H. Print help (this)
 X. Reboot
Ready
] █
```

At the moment, we're using the MCL68k emulator due to it's rather terse size, but this is only an interim fix while we determine the best settings for the bus timing. So the 'J' option will not work.

## Configuration

The user can now set up the EEPROM settings used to configure Buffee at boot. These will also be alterable on the fly -- this is only setting the state of the CPU during boot. You can get there by entering 'E' and then '3' to display and change EEPROM settings. The default options look like this:
```
] 3
Config:
A.  CPU: 
B.  Clock: 7.158 MHz
C.  I$: Enabled
D.  I$ Mask: 0000
E.  D$: Enabled
F.  D$ Mask: 0000
G.  MAPROM Enabled
    000000~07FFFF
    POST:
H.    Long Mem Test Disabled
I.    GreenPAK Test Enabled
J.    Bus Clock Test Enabled
K.    Automap Enabled
L.  MCU CLK: 1000
M.  PMIC Voltage: 1.35
    PJIT Cache:
N.    Block Size: 16384 bytes
O.    Block Count: 1
      Cache Size: 16 kB
X.  Return to previous menu
```
We'll also be adding in options to overclock the 6800 bus which pushes the CIAs to a 9-cycle period instead of 10. We did try other values, but at 8-cycles, they would only respond to every-other-access and were thus quite a bit slower. Still, a 10% speed up on the slowest part of most 68000 machines isn't bad.

You can save these, reset to defaults, and when you reboot, they should be automatically applied. I would strongly not recommend trying to overclock Buffee ***AND*** run from Flash incase your settings brick Buffee. They shouldn't, but *caveat emptor*.

## GPMC Timing

The most important option right now is 4. Test GPMC, which provides you with a sub menu.
```
1. Endian test
2. Dump first 16 words of ROM
3. ROM speed test
4. CIA speed test
5. Agnus read/write
6. CIA read/write
7. Chip RAM read/write
A. Perform all tests and exit
D. Set GPMC timing to default
T. Set GPMC timing
X. Exit to main menu
] █
```

Running all the tests should show something like this when it's working:
```
[GPMC] Endian Check: exec 34.2 (28 Oct 1987)
[GPMC] Bytes NOT swapped
[GPMC] Read words (LE:BE)
[GMPC] $00000000: 1111:1111  f94e:4ef9  fc00:00fc  d200:00d2 
[GMPC] $00000008: 0000:0000  ffff:ffff  2200:0022  0500:0005 
[GMPC] $00000010: 2200:0022  0200:0002  ffff:ffff  ffff:ffff 
[GMPC] $00000018: 7865:6578  6365:6563  3320:2033  2e34:342e 
[GPMC] Test passed and matches
[GPMC] Performing benchmark with ROM
[GPMC] Read 1048576 words in 0.78 s, expecting 0.559s
[GPMC] Read ROM at 2.55 MB/s, expecting 3.58MB/s
[GPMC] Test passed
[GPMC] Performing benchmark with CIA
[GPMC] Read 262144 words in 0.36 s, expecting 0.358 s
[GPMC] Read CIA at 699.10 kB/s, expecting 716kB/s
[GPMC] Test passed
[GPMC] DeniseID: 0xFFFF (OCS)
[GPMC] JOYxDAT Check: 0xa2a3 0xa2a3 (6 avg bit errors)
[GPMC] Test passed
[GPMC] JOYxDAT Check: 0x5253 0x5050 (5 avg bit errors)
[GPMC] Test passed
[GPMC] CIAA Read ok (03 == 03)
[GPMC] CIAA Read ok (00 == 00)
[GPMC] CIAA Timer test (should be 180 ticks)
[GPMC] CIAA timer A Start: 0x00000000
[GPMC] CIAA timer A End: 0x000000b4
[GPMC] CIAA timer A Elapsed: 0
[GPMC] Test passed
[GPMC] Performing RAM test
[GMPC] $00020000: 0000  0101  0202  0303  0404  0505  0606  0707 
[GMPC] $00020008: 0808  0909  0a0a  0b0b  0c0c  0d0d  0e0e  0f0f 
[GMPC] $00020010: 1010  1111  1212  1313  1414  1515  1616  1717 
[GMPC] $00020018: 1818  1919  1a1a  1b1b  1c1c  1d1d  1e1e  1f1f 
[GMPC] $00020020: 2020  2121  2222  2323  2424  2525  2626  2727 
[GMPC] $00020028: 2828  2929  2a2a  2b2b  2c2c  2d2d  2e2e  2f2f 
[GMPC] $00020030: 3030  3131  3232  3333  3434  3535  3636  3737 
[GMPC] $00020038: 3838  3939  3a3a  3b3b  3c3c  3d3d  3e3e  3f3f 
[GMPC] $00020040: 4040  4141  4242  4343  4444  4545  4646  4747 
[GMPC] $00020048: 4848  4949  4a4a  4b4b  4c4c  4d4d  4e4e  4f4f 
[GMPC] $00020050: 5050  5151  5252  5353  5454  5555  5656  5757 
[GMPC] $00020058: 5858  5959  5a5a  5b5b  5c5c  5d5d  5e5e  5f5f 
[GMPC] $00020060: 6060  6161  6262  6363  6464  6565  6666  6767 
[GMPC] $00020068: 6868  6969  6a6a  6b6b  6c6c  6d6d  6e6e  6f6f 
[GMPC] $00020070: 7070  7171  7272  7373  7474  7575  7676  7777 
[GMPC] $00020078: 7878  7979  7a7a  7b7b  7c7c  7d7d  7e7e  7f7f 
[GMPC] $00020080: 8080  8181  8282  8383  8484  8585  8686  8787 
[GMPC] $00020088: 8888  8989  8a8a  8b8b  8c8c  8d8d  8e8e  8f8f 
[GMPC] $00020090: 9090  9191  9292  9393  9494  9595  9696  9797 
[GMPC] $00020098: 9898  9999  9a9a  9b9b  9c9c  9d9d  9e9e  9f9f 
[GMPC] $000200A0: a0a0  a1a1  a2a2  a3a3  a4a4  a5a5  a6a6  a7a7 
[GMPC] $000200A8: a8a8  a9a9  aaaa  abab  acac  adad  aeae  afaf 
[GMPC] $000200B0: b0b0  b1b1  b2b2  b3b3  b4b4  b5b5  b6b6  b7b7 
[GMPC] $000200B8: b8b8  b9b9  baba  bbbb  bcbc  bdbd  bebe  bfbf 
[GMPC] $000200C0: c0c0  c1c1  c2c2  c3c3  c4c4  c5c5  c6c6  c7c7 
[GPMC] Test passed, 0 errors, 0.0%
[GPMC] 7 of 7 tests passed
```

Right now, most commonly, either the RAM or CIA tests fail. This is due to a timing issue we haven't quite corrected yet. We can now use the 'T' option to change the GPMC timing on the fly, to try and dial in a better setting. Right now, the sync behaviour with DTACK and S4 state is controlled by the GreenPAK and isn't adjustable. We may have to change that if we're not able to find good settings.

When changing the timing, we'll be preseted with a timing diagram, the actualy numeric values and then a prompt to change every one. You can simply press enter to accept the default value in brackets. When you are done, it will draw you another cute timing diagram and you can then run more tests.

```
] t
Timing Diagram:
          11111111112222222
012345678901234567890123456
                         *     Access
___                       _
   |_____________________|     AS
        *                      S4 Sync
                           
|_________________________|    DS (Read)
_____                      
     |____________________|    DS (Write)

[GPMC] Cycle Time: 27
[GPMC] Access Time: 25
[GPMC] nCS Timing (ON/OFF): 3/25
[GPMC] nRE Timing (ON/OFF): 0/26
[GPMC] nWE Timing (ON/OFF): 5/26
Set Cycle Time     (0-31) [27]: 
Set Access Time    (0-31) [25]: 21
Set AS On-Time     (0-15) [ 3]: 
Set AS Off-Time    (0-31) [25]: 
Set Read On-Time   (0-15) [ 0]: 
Set Read Off-Time  (0-31) [26]: 
Set Write On-Time  (0-15) [ 5]: 7
Set Write Off-Time (0-31) [26]: 
Timing Diagram:
          11111111112222222
012345678901234567890123456
                     *         Access
___                       _
   |_____________________|     AS
        *                      S4 Sync
                           
|_________________________|    DS (Read)
_______                    
       |__________________|    DS (Write)
       
```

# We Need You!

This is where the testing part comes in. We need to find more reliable timing parameters -- 99% just doesn't cut it. We're working hard on this, but we're welcome to any Beta Buffee owners out there playing around and seeing what works for them. Is NTSC or PAL more forgiving? What about the 1000 versus 500? All great questions!

Anyway, this is a pretty long post already and I think I'll stop here. Next post we're going to be talking about how to run ARM code directly from 68000 code for when 1000 MIPS isn't enough.

TTFN
