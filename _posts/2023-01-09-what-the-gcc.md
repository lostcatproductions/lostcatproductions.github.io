---
layout: post
title: What the GCC?
comments: true
---

Happy New Year! With a new year comes new revelations and wow do I have a doozy -- ARM GCC is just completely broken. I'm not saying it doesn't produce working code but there are cases where it produces obscenely poor performing code.

So the following is a pedagogical example of a direct-threaded interpreter.
```c
#define NEXT goto **ip++

int main(void) {
  static void  *prog[] = {
    &&next1,&&next2,
    &&next1,&&next3,
    &&next1,&&next4,
    &&next1,&&next5,
    &&next1,&&loop
  };
  void **ip=prog;
  int    count = 100000000;
  NEXT;
 next1: NEXT;
 next2: NEXT;
 next3: NEXT;
 next4: NEXT;
 next5: NEXT;
 loop:
  if (count>0) {
    count--;
    ip=prog;
    NEXT;
  }
  exit(0);
}
```
When compiled with -Os, this produces abysmal code. What you're seeing here is the code for each next# statement which is MOSTLY the NEXT statement. This is loading the address to jump to into register R2, branching to .L2 which then copies R2 into the program counter PC. Okay.
```armasm
.L2
        mov     pc, r2    @ indirect register jump
.Lnext1:
        ldr     r2, [r3], #4
        b       .L2
.Lnext2:
        ldr     r2, [r3], #4
        b       .L2
        ...
```
Well, if we compile it with -O3 which should produce faster code and shouldn't produce smaller code, we instead see this:
```armasm
.Lnext1:
        ldr     r2, [r3], #4
        mov     pc, r2    @ indirect register jump
.Lnext2:
        ldr     r2, [r3], #4
        mov     pc, r2    @ indirect register jump
        ...
```
See that? We lost a completely useless branch and the code is smaller. But we're not done. While there is NO compiler option that will improve this further, this is what we SHOULD see with either -Os or -O3:
```armasm
.Lnext1:
        ldr     pc, [r3], #4   @ indirect register jump
.Lnext2:
        ldr     pc, [r3], #4   @ indirect register jump
        ...
```
That's one instruction per NEXT. Ouch. Not only is this now one instruction instead of three, not only would this save unnecessary abuse on the branch predictor, but we also have avoided using a temporarty register: R2. GCC likes to make code-stubs in -Os even when said stubs are only one instruction and this is very bad, but the other problem is that GCC also doesn't understand that on ARM, the program counter is a general purpose register, and with few exceptions can be treated like any other and that means using LDR directly into PC.

Of course the fix here would be to define NEXT with assembly. But this isn't portable anymore and guess what -- it doesn't even work because GCC doesn't see this as a "jump" and will optimize out all the computed goto labels. To date, I've not found a reasonable workaround, portable or not.
```c
#define NEXT asm __volatile("ldr\tpc, [%0], #4" : "+r"(ip))
```

It's with some bit of irony though, that as much as GCC sometimes likes to pointlessly create these little vestigial code stubs, there's also times when it will create a lot of EXCESS code, even with -Os. Here's PJIT's cache routine.
```c
uint32_t* cache_find_entry(uint32_t m68k_addr) {
    uint32_t tag = ((m68k_addr >> 1) >> (BLOCKLEN + INDEXLEN));
    uint32_t idx = ((m68k_addr >> 1) >> BLOCKLEN) & ((1 << INDEXLEN) - 1);
    uint32_t off = ((m68k_addr >> 1) & ((1 << BLOCKLEN) - 1));

    if ((*cache_tags)[idx] != tag) { // MISS!
        __cache_clear(&(*cache_data)[idx][0],
          &(*cache_data)[idx][(1 << BLOCKLEN) - 2]);
        (*cache_tags)[idx] = tag;
    }
    return &(*cache_data)[idx][off];
}
```
Simple enough, right? Well, GCC in it's infinte wisdom basically rolls that return statement into either code path and assembles them both. That is, the end of the function looks like this:
```armasm
        ...
        add     ip, ip, r6
        strh    r5, [ip, #-255] @ movhi
        add     r0, r3, r1, lsl #7
        lsl     r0, r0, #2
        add     r0, r0, #-1627389952
        add     r0, r0, #14680064
        pop     {r4, r5, r6, r7, pc}
.L12:
        add     r0, r3, r1, lsl #7
        lsl     r0, r0, #2
        add     r0, r0, #-1627389952
        add     r0, r0, #14680064
        bx      lr
```
Like seriously? This should be four instructions shorter as there is NO performance gain from poping five registers versus popping four then performing BX LR.
```armasm
        ...
        add     ip, ip, r6
        strh    r5, [ip, #-255] @ movhi
        pop     {r4, r5, r6, r7}
.L12:
        add     r0, r3, r1, lsl #7
        lsl     r0, r0, #2
        add     r0, r0, #-1627389952
        add     r0, r0, #14680064
        bx      lr
```
My best guess is that the optimizer that folds up the stack frame occurs before the subexpression eliminator, so because the end is POP in one and BX LR in another, it cannot merge these into common code blocks. I've found no way of restructuring code to avoid this.

This isn't a unique special case, these problems are endemic to the ARM GCC compiler.

On a happier note, if you're compiling in -Os and want a big code-shrink, add the option `-ftree-vectorize`. This will use the NEON registers for short memory copies and drastically reduces code size -- for example, the compiler can use ldrd and strd to load/store 64-bit chunks. Honestly, if you specify VFP as a compiler option and -Os, it should be enabled by default.

Too bad -Os doesn't ACTUALLY produce the smallest code.
