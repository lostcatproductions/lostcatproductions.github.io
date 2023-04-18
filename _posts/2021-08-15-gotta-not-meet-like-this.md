---
layout: post
title: Beta Buffees Arrived!
---

I'm not one to brag, but when I mess up, I mess up like no one else. But before I start committing seppuku, let me tell you the good news: we got the beta's in and managed to power them up. And wow, do they look gorgeous!

![](https://raw.githubusercontent.com/lostcatproductions/lostcatproductions.github.io/master/images/three_sisters.png)

While we did manage to boot them up and verify that some of the issues we had with the alpha were fixed, we also managed to introduce a couple of more. And a third issue that doesn't really hurt us too much, but it is a pain.

1. the GreenPAK was mounted wrong; in their infintie wisdom, Dialog chose to reverse the pinout between packages. Same number of pins. Completely different order. One is clockwise. One is counter clockwise. Oh my god. This is on me, I should have checked had I thought to, but I've never had a part where the pin outs reversed, so this is a new one for me. Oh well.

2. There are three power rails on the PMIC; AC, USB and BAT. The USB and AC can both take really high voltages, but have a minimum of 4.5V which worried me as these old 68000 machines aren't getting any younger and neither are their power supplies. Getting a solid 5V is rare, so I chose to use the BAT rail instead which operated nicely down to about 2.7V. Except TI didn't make this rail "default on" and needs a push button. What the hell? Anyway, easier fix than #1, just need to short PB_IN to PGOOD.

3. Parts are disappearing and one of them was the 16MB Flash. We had to go with an 8MB one or "not at all". So ... this will be interesting.

So, they're off for some rework and hopefully we get them back by the end of next week. I have to say these shipping delays and parts shortages all on top of dealing with COVID and a new fourth wave is so stressful. But I digress. At least we're still under-budget.

# Other News

Oh what a few months this has been and I do appologise for such a long gap since the last post. We've made excelent progress on Buffee despite all these set backs and have developed a new method of handing the weird world of the variable length 68000 instruction and it's dasterdly sidekick, the extended addressing mode. This has been the one last issue with PJIT that I had yet to solve.

The current method was to bake in the complex decode logic into every instruction that uses it -- and for an addressing mode that's not nearly as often used as say register direct or simple load/store operations, it was eating up a huge amount of the code base, since the "inline" nature of PJIT meant copying the same 20-30 ARM instructions for each opcode. Not efficient!

And not fast either, the instruction routinely had to dig back into 68K memory and to do that, it would need to compute the 68K program code since we don't always track it and ... well, it was a mess.

## Enter multi-word decoding.

So of course the answer was staring me in the face all the time. Our old model shows all these NOP's filling the operand-gaps between opcodes -- so why not simply shuffle them around and have all the literals and extended addressing modes get handled BEFORE we execute the actual instruction.

Most 68000 opcodes are a single word and PJIT must either compile these to a single, inline ARM instruction or branch to a subroutine to handle more complex operations. But there are many 68000 opcodes that take considerably more than one word and PJIT can handle these by having special operators for each word within the whole instruction.

For example

	MOVE.W #$1234, D0
	
Is a multi-word instruction. Immediately after the opcode word, the immediate #$1234 is stored along with it. For PJIT this now gives us two words with which to work with and we'll always have our 1-to-2 relationship between the 68K instruction stream and the ARM instruction stream. But, in most cases, the order will be slightly different.
1. Immediate values are loaded
2. Extension addressing mode is handled
3. The 68000 core operation is executed
	
In cases where there are two extension words (MOVE), this is a little more complex:
1. Source Immediate values are loaded
2. Source Extension addressing mode is handled
3. Destination Immediate values are loaded
4. Destination Extension addressing mode is handled
5. The 68000 core operation is executed

For now we're only dealing with the 68000; the 68020's more complex addressing modes will be dealt with later. So what does this look like?
```m68k
	$303C	MOVE.W IMM,D0
	$1234	IMM
```
The output of this is in reverse though:
```m68k
	MOV R0, #IMM
	BL  MOVE_W_IMM_D0
```
A 32-bit immediate is handled in the same way:
```m68k
	$303C MOVE.W IMM,D0
	$1234 IMM_H
	$5678 IMM_L
```
And as ARM instructions:
```armasm
	MOV  R0, IMM_L
	MOVT R0, IMM_H
	BL   MOVE_L_IMM_D0
```
Okay, sorry if this is going on too much, I'm pretty excited about this. It's going to radically simplify the instruction handlers themselves and make PJIT considerably smaller and ultimtely, more cacheable. This is also going to really help performance and ensure that we can sustain that 1000 MIPS we keep tossing around all willy nilly. Well, until next time!
