### PJIT Pipeline

This is a brief outliåne of the PJIT execution pipeline.

When first encountering a new instruction, PJIT will look up the definition for the opcode which informs PJIT as to the number of ARM opcodes, or length of the instruction and which, if any, extension words are required for the 68K instruction (or it's effective length). When running in emulation mode, the only difference at this point is whether the steps are written to a single-step buffer or into the PJIT cache.

## Pre-Operand State

Extension words are handled externally and prior to the execution of the operand itself. Operations are rearranged so that the opcode is executed last while the extension words are executed first.
```
    16-bit      mov       rD, #immd

    32-bit      mov       rD, #immd-low
                movt      rD, #immd-high

    Extension   bl        <ext-handler>
```
This covers Immediate, Absolute Word or Long, and Address or Program Counter with Index addressing modes as well as preloading the 16-bit displacements for Address or Program Counter with Displacement. In addressing modes using Program Counter, this will already be computed, effectively rendering both Program Counter addressing modes into Absolute Long.

Thus we have a number of addressing mode equivalents
```
	Address, Indexed      (d16,An) or (d8,Xn,An)

    Address, Absolute     (d16,PC) or (d8,Xn,PC) or 
                          (xxx).W or (xxx).L
```
R1 shall hold the source immediate data or address
R2 shall hold the destination immediate address

## Source Compute Effective Address

In opcodes that only want the effective address, that is, LEA and PEA, then we do not perform any actual load, and the data should be the final effective address. In these cases we need to perform arithmetic directly on the address and not use the ea model.

In absolute addressing modes, we do nothing, the data is already in R1.

In indexed addressing modes, the address register will be added within the opcode to R1 or R2.
```
    Memory Register
        ldr        r0, [r12, #offset]
        add        r1, r1, r0
    Processor Register
        add        r1, r1, rX
```

In pre-decrement addressing mode
```
    Memory Register
        ldr        r1, [r12, #offset]
        add        r1,  r1, #-size
        str        r1, [r12, #offset]
    Processor Register
        add        rX, rX, #-size
        mov        r1, rX
```

In post-increment addressing mode
```
    Memory Register
        ldr        r1, [r12, #offset]
        add        r0, r1, #+/-size]!
        str        r0, [r12, #offset]
    Processor Register
        mov        r1, rX
        add        rX, rX, #+/-size
```

Otherwise we simply get the register
```
    Memory Register
        ldr        r1, [r12, #offset]
    Processor Register
        mov        r1, rX
```

We can now skip the next two steps.

## Source Prepare Effective Address

If we're operating only on one operand (e.g., NEG), then the source effective address and data loads are not used -- this is entirely done within the destination effective address. For all other cases, we need to acquire our source data and the first step is determining our (partial) effective address -- we'll leverage ARM's addressing modes to complete the load in the second step.

In immediate addressing modes, nothing is done and there's no 'ea'.

In absolute addressing modes, we do nothing
```
        ea         = [r1]
```

In indexed addressing modes, the address register will be added within the opcode to R1 or R2.
```
    Memory Register
        ldr        r0, [r12, #offset]
        ea         = [r1, r0]
    Processor Register
        ea         = [r1, rX]
```

In post-increment and pre-decrement addressing mode, we can adjust before continuing
```
    Memory Register
        ldr        r1, [r12, #offset]
        ea         = [r1, #+/-size]!
        str        r1, [r12, #offset]
    Processor Register
        ea        = [rX, #+/-size]!
```

Otherwise we simply get the register
```
    Memory Register
        ea        = [r12, #offset]
    Processor Register
        ea        = [rX]
```

## Source Data Load

And then we load the actual data -- note that the source effective address will never be needed beyond this point, so we can reuse it as our data register.

In data register mode:
```
    Memory Register
        B    ldrsb      r1, [r12, #offset]
        W    ldrsh      r1, [r12, #offset]
        L    ldr        r1, [r12, #offset]
    Processor Register, tRR is r1 except long which is rX
        B    sxtb       r1, rX
        W    sxth       r1, rX
        L    (no-op, use rX instead of r1)
```

In address register mode:
```
    Memory Register
        W    ldrsh      r1, [r12, #offset]
        L    ldr        r1, [r12, #offset]
    Processor Register
        W    sxth       r1, rX
        L    (no-op, use rX instead of r1)
```

In immediate modes, r1 should be pre-loaded.

In all other addressing modes
```
    Memory Register
        B    ldrsb      r1, ea
        W    ldrsh      r1, ea
        L    ldr        r1, ea
```

## Destination Effective Address

For destination addressing modes, we need to perform the above two steps for R2. 

In absolute addressing modes, we do nothing and ea points to R2.
```
        ea         = [r2]
```

In indexed addressing modes, the address register will be added within the opcode to R1 or R2.
```
    Memory Register
        ldr        r0, [r12, #offset]
        ea         = [r2, r0]
    Processor Register
        ea         = [r2, rX]
```

In pre-decrement addressing mode we need a separate ea for read and write. Because ea is appended to the operation, we can stick on any extra operations on the end we like to ensure the register get's written back.
```
    Memory Register
        ldr        r2, [r12, #offset]
        ea_r       = [r2, #-size]!
        ea_w       = [r2]; str r2, [r12, #offset]
    Processor Register
        ea_r       = [rX, #-size]!
        ea_w       = [rX]
```

Similarly, in post-increment addressing mode.
```
    Memory Register
        ldr        r2, [r12, #offset]
        ea_r       = [r2]
        ea_w       = [r2, #size]!; str r2, [r12, #offset]
    Processor Register
        ea_r       = [rX]
        ea_w       = [rX, #size]!
```

Otherwise we simply get the register
```
    Memory Register
        ea         = [r12, #offset]
    Processor Register
        ea         = [rX]
```

## Destination Data Load

Note that we do not need to load the destination data if we're completely replacing it (e.g., MOVE). Destination addressing modes need to retain the effective address for the write. In this case, loading the data should be to r0 and not r2.

In data register mode:
```
    Memory Register
        B    ldrsb      r0, [r12, #offset]
        W    ldrsh      r0, [r12, #offset]
        L    ldr        r0, [r12, #offset]
    Processor Register, tRR is r1 except long which is rX
        B    sxtb       r0, rX
        W    sxth       r0, rX
        L    (no-op, use rX instead of r0)
```

In address register mode:
```
    Memory Register
        W    ldrsh      r0, [r12, #offset]
        L    ldr        r0, [r12, #offset]
    Processor Register
        W    sxth       r0, rX
        L    (no-op, use rX instead of r0)
```

In all other addressing modes (is ea_r is null, use ea)
```
    Memory Register
        B    ldrsb      r0, ea_r
        W    ldrsh      r0, ea_r
        L    ldr        r0, ea_r
```

## Execution

Now, with r0 and r1 as our destination and source DATA respectively, we can perform our operation. This would include all logical operations, all arithmetic operations, all shift, multiply and divide operations as well as all move operations.

Any side effects of writing any registers will need to be corrected for here -- iIf the execution needs the X flag, we'll need to restore and save it. If the execution touches the status register, we'll need to check for a privilege violation, etc.

## Destination Write-back

Finally, we then need to store the result from r0 in the destination.

In data register mode:
```
    Memory Register
        B    strb       r0, [r12, #offset]
        W    strh       r0, [r12, #offset]
        L    str        r0, [r12, #offset]
    Processor Register, tRR is r1 except long which is rX
        B    bfi        r0, rX, #0, #8
        W    bfi        r0, rX, #0, #16
        L    (no-op, already used rX instead of r0)
```

In address register mode:
```
    Memory Register
        W    strh       r0, [r12, #offset]
        L    ldr        r0, [r12, #offset]
    Processor Register
        W/L    no-op, already used rX instead of r0)
```

In all other addressing modes (if ea_w is null, use ea)
```
        B    str        r0, ea_w
        W    str        r0, ea_w
        L    str        r0, ea_w
```

## Notes on Special Cases

Imperative operations have no operands and do not use either the source or destination effective address pipelines. Most commonly these are control-flow instructions and must be dealt with differently.

In most cases, branches can exist inline within the cache itself. When thats not possible, we have to use support functions. 
```c
branch_pc(int offset);
jump_pc(int address);
```

Neither of which return.

There is also the case where we jump via the vector table. This is done with the ARM SVC instruction, with the number corresponding to the vector to take. 

