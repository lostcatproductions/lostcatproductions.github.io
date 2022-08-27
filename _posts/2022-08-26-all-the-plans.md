---
layout: post
title: All the Plans of Mice
---

So we have a LOT on the go right now trying to stay ahead of the chip shortage. I'd like to say that it's coming to an end, but we all know that's not true. Some chips have started showing up on sites like DigiKey only to almost immediately disappear within days. It's shocking and it's probably still going to be late 2023 before this settles down.

So we're diversifying!

But before I get too far into all of that, a little mention on some rebranding. Originally, I was sort of against the idea of coopting product numbers, but it really is the best understood way to express design and compatibility (and why Intel eventually trademarked Pentium and ended the numbered processor designation). So hence forth, all chips will be prefixed with BP (Buffee Project) followed by the same numbers as the original product we're "cloning."

And I'd just like to remind everyone that we're making an effort to open source everything we can. There have been some big updates on the PJIT github (https://github.com/nonarkitten/pseudo-jit) as well as the Amiga Replacement Project (https://github.com/nonarkitten/amiga_replacement_project), and there's more to come.

# BP68000

The first up is Buffee Midi, BP68000. The BP68k will be based on a more modern ARM CPU, but runs at about half the clock speed. The balance of this is about 70-80% of the performance of Buffee. It will not have a significant amount of additional RAM, pretty much just enough to cache and create a MapROM. We may have SKUs with some external HyperRAM, but being serialized RAM, it will not be as fast as DDR3 on Buffee.
 - 100% 68000 compatibility (no FPU or MMU though)
 - programmable MapROM feature
 - integrated 4MB L3 cache
 - option to include external RAM (up to 64MB, 100MB/s)
 - about 700 MIPS peak performance

The CPU is an amazing chip that supports bare-metal development out of the box. We're still testing it to see if everything works, but dev boards are in and we're warming up the compilers to start testing. As a life-long embedded developer, it's refreshing to have this. I wish I knew of this chip back in 2019...

Is it the STM32MP1? No. We were strongly considering it and the STM32MP13 (yet to be released) is looking very promising and has 5V tolerance built-in, which is a plus, but had some big difficiencies we didn't like. Namely the same issue the OSD3358 has -- it assumes Linux as the default and getting around that is like pulling teeth. No thanks. Not again.

# BP68040

The original Buffee will be renamed the BP68040. The BP68040 will still be the same design as the current Buffee with a refocus on 68040 compatibility and not 68030+68882. This is largey due to a great amount of simplification in the 68040 that will make development easier on me. Yes, it will still include FPU with 128-bit internal precision, and provide access to the MMU if required.
 - 100% 68040 compatibility
 - programmable MapROM feature 
 - integrated 512MB DDR3 memory and L3 cache (1600MB/s)
 - about 1000 MIPS peak performance

It's still going to be way faster, simply because it has 512MB of 1600MB/s memory. This makes all the difference since running from a 16-bit 7MHz bus can only be sped up so much. The BP68000 may achieve 700 MIPS peak, but it cannot sustain it. The BP68040 running from fast RAM can run pedal-to-the-metal all day.

## BP68040 (Rev B)

We're also making a version of Buffee that blows up the OSD3358 into it's components: the AM3358 CPU, DDR3L memory, power management, etc., but it's been very challenging fitting everything into a DIP 64 form factor AND finding a PCB house that can make a design this complex without it totally breaking the budget. But, this will give use THREE Buffee-tier designs that users will be able to choose from!

# BP8375/BP8372B "Willoe" (Agnus)

With the hand-off from Stephen Leary, we're working hard on finding a way to produce a PLCC compatible Agnus. Code name "Willoe", the current version implements its own RAM and seems to seat well into a normal Agnus socket without any special tooling. We're going to target ECS as a baseline, but it is out intention of eventually providing AGA-over-ECS so that all Amiga's may have AGA. We'll be using original schematics provided by Joe Decuir to implement an as-faithful replication of the original chips.
 - 100% drop-in Agnus compatibility
 - integrated DRAM avoiding the need to populate chip RAM
 - option to upgrade to AGA later

We have more information (WIP) on the Wiki. https://github.com/nonarkitten/amiga_replacement_project/wiki/Willoe-v0.5-(ReAgnus)

# BP8372 "Faith" (Denise), BP8364 "Harmonie" (Paula), BP5719 "Xander" (Gary) and BP8361 "Skinny Willoe"
![Picture of Willoe](https://github.com/lostcatproductions/lostcatproductions.github.io/blob/main/images/Harmonie.png?raw=true)

Which has lead us to the DIP48 common board. There was once a very nice DIP48 board that could be made to drop into just about any socket, but they stopped making this a few years back. With just a small alternation of the power and ground pins we've been able to quickly create boards for Gary, Denise, Paula and "Skinny" Agnus. We already have a perfectly working Gary based on the Xilinx CPLD, but those re becoming hard to get, so we're making multiple designs.
 - common-ish board for Denise, "Skinny" Agnus, Paula and Gary
 - cool "flip-chip" design for more chip-like look
 - possibility of including up to 16MB pSRAM
 - option to upgrade to AGA later

# BP8520 "Lennie" (CIA) and more...
![Picture of Willoe](https://github.com/lostcatproductions/lostcatproductions.github.io/blob/main/images/Lennie.png?raw=true)

Which led us to a DIP40 deisgn, since that's just a matter of cropping off a few pins. This can become so many things, from the Amiga CIAs to the very CPU used by the Commodore 64. Yeah, FPGA 6502's have been done, but I thought it would be fun to make our own as well. So we'll have a "Buffee Mini" as well.
 - common-ish board for 6502, 6510, 65816, 8086, Z80, VIC-II, CIA, VIA, etc...
 - cool "flip-chip" design for more chip-like look
 - ability to vastly exceed the performance of original CPUs
 - possibility of including up to 16MB pSRAM

Oh and we're making a DIP28 of that too. Oh, hello SID. And 28-pin and 40-pin ROMs. And more. These three foot prints actually cover a shocking number of parts from our beloved 8-bit and 16-bit machines.

# AGA over ECS

It's fair to point out that AGA-over-ECS will require a custom Gary, Agnus and Denise firmare. Yes, it's possible to push that much data around with our own RAM and no, it woulsn't be possible using stock motherboard RAM (in some cases, Commodore used pointlessly faster RAM in later ECS boards simply due to availablity, but that's not common enough). With just Denise and Agnus, it may be possible to get most of AGA except for the CPU-to-RAM bandwidth which may be a problem for some games. With DMA caching, we can actually relieve a LOT of DMA access making chip RAM about as fast as it could be and allowing AGA to compete with RTG.

Sure, AGA's not for everyone, so we're also going to give ECS the option of enabing DMA caching as well. This means you have all the wonders and warts of ECS, with all the spare bandwidth you could want, granting near Amiga 3000 chip RAM performance in the Amiga 500 and 2000.

# Really Embracing Open Source

But that's not the end. All of these designs will be using the iCE FPGA. What does this mean? It means an open source tool chain as well! No more vendor lock in, anyone can download and build the code for these chips without hacks, cracks and virtual machines. All the software and firmware will be made available here.

# Can We Make an All-New Amiga Yet?

Oh we're close. If only someone made a DIP18 DRAM clone...
