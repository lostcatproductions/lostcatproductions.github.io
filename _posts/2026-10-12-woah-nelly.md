---
layout: post
title: Whoa Nelly
---

# PJIT vs Apple: I Was Wrong

Back in 2024 I wrote [PJIT vs Apple](https://www.buffee.ca/pjit-vs-apple/), where I explained how Apple's 68K emulator for PowerPC worked, how Eric Traut's "Dynamic Recompiler" improved on Gary Davidian's original, and how much PJIT had in common with it. Since then I've taken that emulator apart, all of it, instruction by instruction, and I have some crow to eat. Nearly everything I said about the Dynamic Recompiler (DR) was wrong. WRONG wrong.

There are really three stories here -- what I thought the DR did, what Apple's patent says it did, and what's actually sitting in the ROM -- and no two of them agree. So let's go through them.

## What I Said

This is the listing from the old post, showing how the DR supposedly handled `ADDQ.W #4,D2`:

```
    DR Cache
        lis     data, 0x0004
        bl      dr_add_w_shifted_data_d2
        lhau    prefetch_data, 2(pc)
        bt      cr_sp_event, handle_special_event

    DR Semantic Routines
    dr_add_w_shifted_data_d2:
        slwi    immed_data, d2, 16
        addco.  data, data, immed_data
        rlwimi  d2, data, 16, 16, 31
        mfxer   ccr_x
        blr
```

The story that went with it was that the DR cache is organized like a CPU cache. Each cache line holds 32 entries, each entry is exactly four PowerPC instructions, and there's one entry for every two bytes of 68K code. The first instruction is copied from the interpreter's dispatch table, the second calls a "DR Semantic Routine" found through a 128KB "DR Offset Table", the third does the 68K prefetch and the fourth checks for interrupts.

In my head this was code-in-cache. The 68K program counter picks the line, picks the entry, and off you go -- the PowerPC just runs straight down the cache, entry after entry and line after line, the same way a real CPU runs out of its instruction cache. It made SO much sense to me that I never questioned it.

## Fiction #1: Mine

All of that came from one document -- [US Patent 5,790,825](https://patents.google.com/patent/US5790825A/en), "Method for Emulating Guest Instructions on a Host Computer Through Dynamic Recompilation of Host Instructions" (catchy!) by Eric Traut. I read the patent, looked at the figures, and assumed that's what was in the ROM.

The first problem is that I didn't even read the patent properly.

The patent's cache does NOT run from line to line. The last entry of every line ends with a branch to a routine called `dispatch_next_pc`, which checks for interrupts, works out which line the new PC belongs in, compares a tag to see whether that line still holds the right code, and recompiles the line if it doesn't. Every jump, trap and exception goes through that same routine, and so does every branch that leaves the line. So at the very best you get 64 bytes of straight-line 68K code before you're back in the dispatcher.

"Running straight from cache" was my invention. It's what I would have done, so I assumed it's what Apple did.

It's also a HUNGRY design. Every two bytes of 68K get sixteen bytes of PowerPC -- eight times the size -- and that goes for every word in the line, whether it's an opcode, an extension word or a string somebody left in the middle of their code.

## Fiction #2: The Patent's

Here's the bigger problem. None of it is in the ROM.

No DR Offset Table. No DR Semantic Routines -- there's no `dr_add_w_shifted_data_d2` or anything like it. No tag store, no cache lines, no `dispatch_next_pc`. I went through 23 ROM images, from the Quadra 840AV to the PowerBook G3 Series, and this is what's in them:

- The NuBus Power Macs (6100, 7100, 8100) have Gary Davidian's interpreter and nothing else. No DR at all.
- The DR shows up with the first PCI Power Macs (7200, 7500, 8500, 9500) and it is ALREADY a completely different design from the patent.
- Everything after that is the same design with more polish.

Those PCI machines went on sale in the middle of 1995. The patent application went in that November. So unless those ROM dumps aren't what they say they are, Apple was already shipping something else when the paperwork was filed.

What was the patent's design, then? An early prototype that got replaced? Something that only ever existed on paper? I don't know, and I'm done guessing about this emulator. All I can say is that it's not in anything I can find, and I looked pretty hard.

To be fair to the patent, its numbers aren't entirely made up. It says `CLR.B D5` takes 4 cycles interpreted and 2 recompiled, and when I count the real interpreter the way the patent counts, I get 4. I even built a little model of the patent's design to see how it WOULD have done (more on that below). It's just that `CLR.B D5` is the best case there is; an instruction so short it doesn't need a semantic routine at all.

## What's Really In There

A quick cheat sheet for anyone as allergic to PowerPC assembly as I am. The names (`d2`, `pc`, `data`, `addr`) are all just PowerPC registers -- every 68K register lives in one. Then:

- `addco.` is an add. The `c`, the `o` and the dot mean "and set the carry, the overflow and the condition flags".
- `rlwimi` is rotate-and-insert, which is a bitfield insert (`bfi` for the ARM folks).
- `lhau` loads a 16-bit word AND bumps the pointer, so `lhau prefetch_data, 2(pc)` is "PC += 2 and fetch" in one instruction.
- `mfxer` copies the carry out to a register, which is how the 68K's X flag gets saved.
- `bl` and `blr` are call and return.

### The Interpreter Was Never the Problem

The one listing I got right in the old post was the interpreter, because that one came from real code:

```
    Interpretive Dispatch Table
        lis     data, 0x0004
        b       add_w_shifted_data_d2

    Interpretive Semantic Routines
    add_w_shifted_data_d2:
        rlwimi  disp_table, prefetch_data, 3, 13, 28   ; NEXT opcode x 8 -> table address
        slwi    immed_data, d2, 16
        mtlr    disp_table                             ; ...which is where we're going
        addco.  data, data, immed_data
        lhau    prefetch_data, 2(pc)                   ; PC += 2, prefetch
        mfxer   ccr_x
        rlwimi  d2, data, 16, 16, 31
        bflr+   cr_special_event_pending               ; go! (unless an interrupt is pending)
        b       process_special_event

    Interpretive Code Path: 10 instructions (normally)
```

What I got wrong was the commentary. I said the dispatch table "would jump to a short sequence which would simply load the opcode and then jump again" and that Eric Traut "solved this with the cache". Look at that routine again. Four of its instructions have nothing to do with adding -- the first `rlwimi`, the `mtlr`, the `lhau` and the `bflr+` are decoding and dispatching the NEXT 68K instruction, woven in between the real work so the PowerPC can overlap them. There's no fetch loop to go back to. Nearly every routine in the interpreter ends like this. (There is a shared `fetch` routine, but it's only for the few thousand opcodes that finish their whole job in the dispatch table's first instruction.)

I also said Davidian's emulator "wasn't terribly fast". Sorry, Gary. It's an astonishingly tight piece of work, and between the first ROM with a DR in it and the last, his instruction routines don't change by a single instruction.

### The Real Dynamic Recompiler

The real DR doesn't have cache lines. It has BLOCKS.

When the emulator takes a branch, a jump or a return to somewhere that hasn't been translated yet, the DR starts at that address and makes three passes:

1. **Decode.** Walk forward through the 68K code, working out how long each instruction is and which flags it reads and writes, until something ends the block -- a change of flow, an instruction the DR doesn't translate, or 120 bytes of 68K code.
2. **Liveness.** Walk BACKWARD through the block working out which condition codes anybody actually reads. Flags that get overwritten before they're read are dead.
3. **Emit.** Walk forward again and write out PowerPC code for each instruction, using a cheaper version wherever the flags are dead.

So what does our `ADDQ.W #4,D2` turn into? If something later on needs the flags:

```
    DR Block
        lis     addr, 0x0004
        slwi    rtn_addr, d2, 16
        addco.  addr, addr, rtn_addr
        rlwimi  d2, addr, 16, 16, 31
        mfxer   ccr_x

    Dynamically Recompiled Code Path: 5 instructions
```

Those are the same five instructions that do the work in the interpreter, and nothing else. No call, no return, no prefetch, no interrupt check. And if the next instruction stomps on the flags anyway (I followed it with an `ADD.L D1,D1`):

```
    DR Block
        addi    addr, d2, 4
        rlwimi  d2, addr, 0, 16, 31

    Dynamically Recompiled Code Path: 2 instructions
```

The interpreter takes 10 instructions for this. The patent's design would have taken 9. The real DR takes 5, or 2.

The rest of the machinery is just as different from what I described:

- **Finding a block.** There IS something cache-shaped in there -- a 64KB lookup table of 2,048 sets with four entries each, indexed by bits of the 68K PC and kept in most-recently-used order. But it holds POINTERS to translated blocks, not code, and it's only consulted when the DR doesn't already have a direct route to where it's going.
- **Chaining.** A block that ends by going to a known address ends with a call into the DR's run-time. The first time through, the run-time finds (or translates) the target and then PATCHES that call into a direct branch. From then on the two blocks run back to back and the run-time never hears about it.
- **Returns.** An `RTS` can't be patched since it goes somewhere different every time, so each one gets a one-entry cache sitting right behind it in the code -- the last 68K address it returned to, and where that address's translation lives.
- **Interrupts.** The patent checks for interrupts on nearly every instruction. The code the real DR generates checks on backward branches, and that's it. A loop gets one check per trip around.
- **The PC and the prefetch** aren't kept up to date inside a block at all. The 68K PC only gets fixed up on the way out.
- **Compare and branch.** A `CMP.L D1,D0` followed by a `BHI` becomes one PowerPC compare and one PowerPC branch -- two instructions for the pair -- when nothing else needs the flags.
- **Anything hard** (rotates, bit fields, BCD, 32-bit multiply and divide, anything touching the SR) simply ends the block. The interpreter runs that one instruction and the DR picks up again afterwards. 41,911 of the 65,536 possible opcodes get translated.
- **Self-modifying code** is not detected. At all. Translations get thrown out when 68K code asks for a cache flush (`CINV` and `CPUSH`, plus a private Apple opcode for flushing a range), which is exactly the deal you get from a real 68040.

Here's a whole loop, straight out of the real translator. All I've done is swap the offsets for labels and add the comments:

```
    68K
            MOVEQ   #100,D0
            MOVEQ   #0,D1
    loop:   ADD.L   D0,D1
            SUBQ.L  #1,D0
            BNE.S   loop
            RTS

    DR Block
    bail:   b       dr_exit_to_interpreter
    poll:   bf+     cr_special_event_pending, loop   ; no interrupt? carry on
            addi    pc, pc, 4                        ; otherwise fix up the 68K PC
            b       bail                             ; ...and let the interpreter have it
    entry:  li      d0, 100                          ; MOVEQ #100,D0 (flags are dead)
            addco.  d1, zero, zero                   ; MOVEQ #0,D1
    loop:   addco.  d1, d1, d0                       ; ADD.L D0,D1
            li      addr, 1                          ; SUBQ.L #1,D0
            subfco. d0, addr, d0
            subfe   addr, zero, zero                 ;   (these two save X)
            addc    ccr_x, addr, addr
            bne     poll                             ; BNE.S loop
            lwz     pc, 0(a7)                        ; RTS
            addi    a7, a7, 4
            bl      dr_indirect_exit
            .long   0, 0                             ; the RTS's one-entry cache

    Dynamically Recompiled Code Path: 7 instructions per loop
```

Seven PowerPC instructions per trip around a three-instruction 68K loop, and that includes the interrupt check. The interpreter takes 26.

There are even a few hand-picked specials. The classic string copy loop `MOVE.B (A1)+,(A0)+` and a `BNE.S` back to itself gets recognized as a pair:

```
    DR Block
    loop:   lbz     addr, 0(a1)
            addi    a1, a1, 1
            cmpwi   cr7, addr, 0
            stb     addr, 0(a0)
            addi    a0, a0, 1
            bne     cr7, poll                        ; poll, then back to loop
```

And my favourite, a 68K `NOP` gets translated into a PowerPC `eieio`. Yes, that's a real instruction.

## The Numbers

PowerPC instructions executed per trip around four little 68K loops:

| 68K loop | Interpreter | Patent's design | What shipped |
| --- |:---:|:---:|:---:|
| `ADD.L` / `SUBQ.L` / `BNE` | 26 | 21 | 7 |
| `MOVE.L (A0)+` / `ADD.L` / `MOVE.L (A1)+` / `DBRA` | 38 | 27 | 13 |
| `BSR` / `ADDQ.L` / `ADD.L` / `RTS` / `DBRA` | 49 | 37 | 33 |
| `CMP.L` / `BLE` / `ADDQ.L` / `SUBQ.L` / `BNE` | 47 | 36 | 9 |

The first and last columns are counted from the real code running in a simulator. The middle one is my model of the patent's design, since there's no real thing to measure, so treat it as an estimate. And these are instruction counts, not cycle-accurate timings.

In a nut-shell, the patent's design saves the dispatch and not much else, which gets you about a quarter fewer instructions than the interpreter. The real DR saves the dispatch, the prefetch, the interrupt check, the call and return AND the dead flags. The one place it doesn't run away with it is call-heavy code, because every `RTS` has to go out through the run-time.

In the old post I said the DR was "in some cases, doubling its speed". For the design I described that was generous. For the one that shipped it's an undersell -- on tight loops it's executing a third to a fifth of the instructions.

Is it free? Nope. The patent says its whole recompilation process takes "fewer than twenty PowerPC instructions in the average case". The real translator burns 80 to 115 PowerPC instructions for every 68K instruction it translates. Apple decided that was worth paying, and by my math it pays for itself once a loop has gone around somewhere between ten and thirty times -- code that's all calls and returns takes a lot longer to pay it back.

## Three Names in the ROM

There are two credits buried in the emulator's code. One says `680x0 Emulation written by Gary Davidian`. The other one grew a name at a time:

- **The first PCI Power Macs (1995)** -- "Dynamic Recompilation by Eric Traut". Blocks, three passes, the lookup table, chaining. All there from day one.
- **Performa 6400, PowerBook 1400, Power Mac 6500 and the G3s** -- "...tuned by Bob Abeles". The lookup table doubles in size, the compare-and-branch trick arrives, a few more instructions get translated (`ADDX` and the 16-bit divides), cache flushes are done a bit at a time so they can't hold off an interrupt, and the interrupt check comes off the forward branches.
- **The "Mac OS ROM" file that later Macs load from disk** -- "...hacked by Murph". As far as the recompiler itself goes, four cache-flush instructions changed.

So the design I described wasn't an early version that got tuned into something else. As far as the ROMs go, it was never there.

## So What About PJIT?

This is the part that stings. In the old post I wrote that "PJIT and Apple's DR share a LOT in common". They don't. PJIT shares a lot in common with Apple's PATENT -- or what I imagined the patent to be -- fixed-size slots in the cache, a branch-and-link to a semantic routine for each opcode, an offset table so the slots can be filled in cheaply. I even reorganized PJIT's tables around what I read there. The thing Apple actually shipped doesn't do any of that.

That doesn't make the patent's design dumb. Translating is nearly free and it never has to think -- quite the contrast to the 80 to 115 instructions to convert one, single 68K instruction. But the numbers above say where the real DR gets its speed from:

- no call and return for every opcode
- no prefetch and no PC bookkeeping inside a block
- no interrupt check on every instruction
- no computing flags that nobody reads
- compares and branches fused into one native compare and branch

PJIT already dodges a couple of those -- it never did the prefetch, and interrupts are handled by yanking the link register instead of polling. The rest is food for thought, and I'll have more to say when I'm back on the ARM side of things.

## Fiction vs Reality

The whole disassembly is up on GitHub at [apple_68lc040_dyarec](https://github.com/nonarkitten/apple_68lc040_dyarec). The source is labelled and commented from top to bottom, there are fifteen chapters of documentation, and it reassembles into a byte-for-byte identical copy of Apple's binary -- so if you think I've got it wrong AGAIN, you can check. Please do. There's a PowerPC simulator in there that runs the emulator, which is where the DR listings and the instruction counts in this post came from. (I even found a few genuine bugs in Apple's code along the way. They're written up in the docs.)

A big thank you to Elliot Nunn, whose [powermac-rom](https://github.com/elliotnunn/powermac-rom) project is where the emulator binary came from, and whose archives of Apple's kernel and ROM sources put real names on a lot of things.

I'm leaving the old post up, with a note at the top pointing here. Being wrong in public is part of the deal.

And thanks, as always, for reading.