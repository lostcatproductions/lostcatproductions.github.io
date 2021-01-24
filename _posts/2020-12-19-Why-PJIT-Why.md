---
layout: post
title: Why PJIT, Why?
---

So I've had a few people ask why PJIT and not leverage something else like Emu68.
Is there really room in the fairly small market for two 68000 to ARM JIT engines for the Amiga?
Well, of course I think so and here's why.

## What is Emu68

Emu68 is the very excellent bare-metal (meaning that it runs without the benefit of any operating system)
traditional JIT developed by [Michal Schulz](https://github.com/michalsc/Emu68)
and already shows some very credible benchmark scores and is a reasonably complete at this point. As with all
traditional JIT compilers, Emu68 takes the 'entry point' (where code is starting) and translates every 68000
opcode into ARM instructions up to some end point (usually a branch).

While I think Emu68 is a great project, I do think there are a couple of issues. These aren't issues specific to Emu68, they're common
with all modern JIT implementations and to date, they've been regarded as 'unavoidable'.

1. JIT lag (jitter)
2. High memory requirement

## Jitter

Jitter is caused when the JIT code needs to be created the first time before it can be executed. It is so expensive that most modern JITs will avoid it on the first pass of code. On Emu68,
the jitter shows about a 20~25% performance difference between the self-reported SysInfo scores and the "wall clock" benchmark that includes the time to create the block.

PJIT avoids jitter by running each opcode as we decode it much like an interpreter. This only works in PJIT because all opcodes are just subroutines.
Being subroutines does mean that there's one extra cycle of overhead per opcode in 'ideal circumstances'. That is, a straight-run of Emu68 code will be 25~33% faster than 
PJIT. 

My expectation is that the near-elimination of that jitter will make up the difference.

## Memory

In Emu68, a single 16-bit 68000 opcode can take several 32-bit ARM instructions to execute; if every opcode took just three, then that's a 600% increase in code
size, not including the extra bit of code to enter and exit each block and the possibility of the same block of code needing to be translated several times from slightly
different entry points.

In PJIT, every 16-bit 68000 opcode takes one and only one 32-bit ARM instruction; it would only ever need to be recompiled on a cache miss (or flush). 
This has a second benefit; not only does PJIT use much less memory, it also means that when jumping to a new address, PJIT knows exactly where in
the cache to jump to. This means that control logic has the potential to be quicker.

Sure, RAM is cheap, but why waste it on giant JIT caches?

## Elegance

In the end, modern JIT compilers become vast quagmires of corner cases, full of micro-optmizations and complex scheduling to get the maximum performance. They have
become very complex and unweildly. To perform well, Emu68 may need multipass optimizations and even add an interpreter to avoid the performance hit of run-once code -- right 
now we don't know as little more than tiny benchmarks have been run on it.

PJIT is tiny in comparison; in fact, the core logic consists of only a few dozen lines of code. The bulk of PJIT is the 68K opcode table code generator which
is handled at compile-time, not run-time. All the corner cases are handled; we're no worse than an interpreter already (and may be quite a bit faster) and we're
approaching peak performance of "ideal" JIT code when acounting for jitter.

What we end up with is a processor that's **smoother** in operation and more like a real 68000. It's also a completely new way of doing JIT, and that alone 
is a good enough reason for me, even if we're never as fast as Emu68.
