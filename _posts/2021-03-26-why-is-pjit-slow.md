---
layout: post
title: Performance Series Part 2 -- Why is PJIT Slow?
---

In the last part of the series we put PJIT up against the state-of-the-art 68000 emulator for ARM that used the traditional interpreter style of emulation, Cyclone 68000, and found PJIT to be rather compelling. But one cannot ignore the raw performance of a true JIT.

Generally, there are three reasons why a true JIT is going to be faster than PJIT:
- lack of threading overhead
- register colouring
- additional optimizations

## Lack of Threading Overhead

We left off the last series with this little bit of code for a MOVE.B:
```
    bl #offs                 ;@ to get here...            

;@ ---------- [1000] move.b d0, d0 uses Op1000 ----------
Op1000:

    ldrb r2,[r7,#4]          ;@ load byte from D1
    strb r2,[r7]             ;@ store in D0

    bx lr                    ;@ and return
```
You can see that the part of this that actually performs the operation (the LDRB and STRB) make up only half the code as there are two branches surrounding us -- the branch to get here and the branch to get back. When a traditional JIT is compiling a block, there are no branches, so instead, the ARM instructions are simply concatenated together before then block is executed. 

Now, because of branch prediction and return caching, neither of these will cost us a pipeline stall, but they're still instructions and they still have overhead and a traditional JIT should execute this about twice as fast as PJIT.

## Register Colouring

Because each opcode for PJIT is a fully pre-compiled subroutine, we have to design a static allocation for each of the 68000's registers, of which there are sixteen. Unfortunately, there are more 68K registers than there are available ARM registers.

The solution traditional JITs use is called Register Colouring -- or dynamic register allocation. That is, there is no direct association between the ARM and 68K registers and instead the JIT will allocate ARM registers as required. Let's look at a little subroutine:
```
    addl d1,d0       ;@ find two free ARM registers; R0 and R1 then add
    asrl #1,d0       ;@ divide by two and store back to D0
    rts              ;@ were done, lets go home
```
When PJIT sees this, it creates a branch to the subroutine to three routines; the specific ADD.L, the ASR.L and then the RTS. These look something like this:
```
    ...
    bl ADDL_D1_D0
    bl ASR_L
    bl RTS
    ...

ADDL_D1_D0:
    ldr   r3, [r7]
    ldr   r2, [r7, #4]
    adds  r3, r3, r2
    str   r3, [r7]
    bx    lr

ASR_L:
    ldr   r3, [r7,]
    lsrs  r3, r3, #1
    str   r3, [r7]
    bx    lr

RTS:
    ldr   r0, [r13], #4
    bl    reenter
```
These 14 instructions would take about eight cycles to execute and aren't ideal. There's a lot of load and store pressure here and we'd better have that state in some zero-cycle SRAM! However, regular JIT would do something more like this:
```
    ...
    ldr   r3, [r7]
    ldr   r2, [r7, #4]
    adds  r3, r3, r2
    lsrs  r3, r3, #1
    str   r3, [r7]
    ldr   r0, [r13], #4
    bx    lr
```
This takes about four cycles and ought to be nearly twice as fast as PJIT. Not only are we saving the extra branches here, but we're also performing a sort-of automatic peephole optimization by not storing and reloading the registers from memory!

## Additional Optimizations

This is truely a well with no bottom and different architectures benefit from different optimizations, but great care has to be taken as the performance impact to the system to analyse and optimise code may be greater than the benefit of slightly faster actual execution. I won't go into too much detail here, but the common optimizations I'm aware of are: subroutine inlining, constant folding and loop simplificaion.

### Subroutine Inlining

When jumping to an absolute addresses, the JIT may follow the jump and create a super-block that inlines the subroutine. This avoids leaving the JIT code and re-entering back in the main loop. But like many things with JIT, this not only means potentially re-JIT-ing the same code over and over again for every unique caller, but also means yet another way JIT code can use memory excessively.

### Constant Folding

This simply takes constant values loaded into the 68K "scratch registers" that are only used once as an offset for another memory action. It can eliminate the entire opcode by essentially deferring the immediate value to when it's needed later and is one of the few ways JIT can save memory.

### Loop Simplification

Normally each JIT block will have some entry and exit code. At the end of the block, the CPU will return to the main loop, reevaluate the program counter and re-search the cache for the block to execute or compile a new one if necessary. For small, closed loops, this is not optimal, so a simple fix is when the branch returns to the begining of the current block, to do so on only exit the block when the loop exits.

## Conclusions

So traditional JIT clearly looks a lot faster that PJIT, easily up to twice as fast! In fact, Emu68 can get very close to 1MIPS per MHz on JIT compiled code, which is absolutely amazing! But there are some things I've completely glossed over here that level the playing field quite a bit.
