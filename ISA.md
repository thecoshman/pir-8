Unless stated otherwise, bits are always represented from MSB to LSB (reading left to right) and multi-bytes sequences are big-endian.
So, a jump instruction followed by a two byte address would have the following sequence of bytes "jump", "high address byte", "low address byte".

# Registers

All the registers will start with an initial value of `0x0`.

## Special Purpose Registers

There are some special purpose registers that you cannot directly read/write from, these are used by the CPU for its internal state.

There are three 16-bit registers for holding significant memory addresses and a single 8-bit register.

Name            | Short | Bits | Description
----------------|-------|------|------------
Program Counter | PC    |  16  | Address of the next instruction to be fetched
Stack Pointer   | SP    |  16  | Current address of the stack (detailed later)
Memory Address  | ADR   |  16  | Address saved for use during certain instructions
Instruction     | INS   |   8  | Instruction currently being executed

The address bus is controlled by either PC, SP or ADR.
As the CPU is reading an instruction from system memory, the value of the PC will be used, for some instructions though, such as JUMP or STACK, it is value of ADR or SP (respectively) that is used.

## General Purpose Registers

There are eight 8-bit General Purpose registers, each has an internal address for use within the CPU, instructions like MOVE and LOAD can use these addresses.

The first five registers have some special functionality, as described, the last three have no special functionality.
The last four registers can also be used with the stack.

Address | Letter | Description
--------|--------|------------
000     | F      | Flag register (can also be used to get a zero value)
001     | S      | Output of the ALU - ALU operations will overwrite any value stored
010     | X      | Input to ALU (Only input for unary operations)
011     | Y      | Second input for ALU
100     | A      | Port number for PORT instruction
101     | B      |
110     | C      |
111     | D      |

## Flag register

The flag register can be be read and written to as a general purpose register, though keep in mind that ALU and Compare instructions can effect the value of the flags.

Not all of the bits have a specified role (yet), though the CLRF operation will still clear them.
A value of `1` denotes the flag as 'set', whilst a value of `0` denotes  the flag is 'unset'.
Below is a description of what each bit in the flag register denotes.

Bit | Letter | Description
----|--------|------------
0   | Z      | Zero flag
1   | C      | Carry flag
2   | P      | Parity (even number of set bits)
3   | E      | Equals flag
4   | G      | Greater than
5   |        |
6   |        |
7   |        |

# Instructions

Instructions will increase the PC by one, unless otherwise stated.
The PC is incremented as the instruction is loaded from RAM.
An instruction is a single byte, and can include some following immediate values purely for data.

It is possible that the PC will overflow and wrap around as you load an instruction, there is no hardware level protection or detection if this happens.
An example of how this can occur is if you perform a jump to `0xFFFF`;
as the instruction at `0xFFFF` is loaded, the PC would be incremented to `0x10000`, but as it's only 16 bits wide, it becomes just `0x0000`.

The 'Bit Mask' shows a pattern which denotes an instruction or group of instructions, the letters denoting where any value can be used and still be considered part of the same instruction.
The 'name' is for either a group or single instruction.
'Count' is how many of the 256 possible instructions are used by that bit pattern;
HALT for example is exactly one instruction, whilst MOVE is effectively 64 possible combinations [this was added to help me keep track of how many operations I've defined, it should add up to 256].

Bit Mask  | Name | Count | Description
----------|------|-------|------------
000X XXXX | LOAD |    32 | Load, see section below
0010 0XXX | JUMP |     8 | Jump, see section below
0010 1RRR | SAVE |     8 | Store value in register `RRR` in memory addressed by `ADR`
0011 XXXX | ALU  |    16 | ALU based operations, see section below
01QQ QRRR | MOVE |    64 | Move a value from register `QQQ` to register `RRR`
10XX XXXX |      |    64 | Reserved
1100 XXXX |      |    16 | Reserved
1101 0XXX |      |     8 | Reserved
1101 10XX | MADR |     4 | Move a value to/from the ADR register, see section below
1101 11XX |      |     4 | Reserved
1110 XXXX | PORT |    16 | Perform I/O, see section below
1111 0RRR | COMP |     8 | Compare register S with register `RRR`, see section below
1111 10XX | STCK |     4 | Stack manipulation, see section below
1111 110X |      |     2 | Reserved
1111 1110 | CLRF |     1 | Clear the F register, by setting it to `0000 0000`
1111 1111 | HALT |     1 | Stop the CPU from doing any more execution

## LOAD

There are various types of load instruction, all under the same `000X XXXX` pattern.
They may further increment the PC, as described in their relevant sub-sections.
Some of the potential instruction patterns are simply reserved.

The following table is a break down of possible LOAD instructions.
If you consider load instruction pattern as `000M XXXX`, the `M` bit is `0` for single byte loads and `1` for multi-byte loads.

Bit Mask | Load Type      | Description
---------|----------------|------------
  0 0RRR | Byte immediate | Load the the next byte into register `RRR`
  0 1RRR | From memory    | Load the byte address by `ADR` into register `RRR`
  1 00XX | Wide immediate | Load the the next two bytes into a register pair
  1 01XX |                | Reserved
  1 1XXX |                | Reserved

### Wide Immediate

When performing a wide immediate load, two bytes are read from memory into a pair of registers, as indicated by the two bits `RR`.
The first byte read (after the actual LOAD instruction itself) is loaded as the high byte, the second is the low byte;
registers A, C and X are considered the 'high byte' for this purpose;
if loading into ADR, the high/low bytes are loaded as the high/low byte of the address for ADR.

RR | Registers
---|----------
00 | A and B
01 | C and D
10 | X and Y
11 | ADR

The PC is incremented twice more.

### Byte Immediate

When performing a byte immediate load, the next byte (after this instruction) is read from memory into a register, as indicated by the bits `RRR`.

The PC is incremented once more.

### From memory

For loading from memory, the memory address to load from should have already been set in the ADR register and the instruction says what register to load into.

The PC is not incremented any further.

## PORT - I/O

The PORT instruction in the form `1110 DRRR` will perform I/O on the port specified in register A.

The `D` bit specifies the direction;
`1` for reading in from the port (PORT IN) and `0` for writing out to the port (PORT OUT).
The `RRR` bits specify the register to write to (IN from the selected device) or read from (OUT to the selected device).

It is possible to read a value form a selected device to the A register, via `1110 1100`, which then changes the selected device.
This will be stable though, and would require a further such operation in order to for the device selection to change once more.
The value is latched on the clock edge.

**NB:** It is currently planned the data is be latched on the rising edge.
The clock signal will also be exposed out for I/O device, though it may only be done so when the CPU is expecting to read/write.
The control line to signify to I/O devices that they should read/write would also be exposed, but under what timings is not yet determined.

## COMP - Compare

The compare instruction will compare the S register with a register selected by bits `RRR`.
It will set the Zero and Parity flag based on the value of the S register;
the Zero flag if all the bits are zero, Parity if the number of set bits is even.
Compare will set the Equal flag if the two registers have the same bit pattern.
The Greater than flag is set if S is greater than the second register.
Note that when doing a compare signed/unsigned is not taken into account, the two registers are treated as if they contain two unsigned values.

**NB:** This might change to instead compare just the X and Y register.

## ALU Instructions

Any CPU instruction of the pattern `0011 FFFF` will invoke some function of the ALU.
The four bits `FFFF` are the actual operation being performed by the ALU.
The registers X and Y are used as inputs to the ALU (only X for unary operations), and the S register is used to store the result.

ALU operations will also update the F register as noted.
All will set the ZERO flag if the output (register S) is `0000 0000`.
All will set the PARITY flag if the number of high bits are even in the S register.
The ADD, SUB, ADDC, and SUBC operations will set the carry bit in F to carry out value from the adders.

FFFF | Name | Count | Description
-----|------|-------|------------
0000 | ADD  |   1   | Addition of register X and register Y
0001 | SUB  |   1   | Subtraction of register Y from register X (X-Y)
0010 | ADDC |   1   | Addition of register X and register Y, using the carry bit from F (X+Y+C)
0011 | SUBC |   1   | Subtraction of register Y from register X (X-Y), using the carry bit from F (X-Y-C)
0100 |  OR  |   1   | Bitwise OR
0101 | XOR  |   1   | Bitwise XOR
0110 | AND  |   1   | Bitwise AND
0111 | NOT  |   1   | Bitwise NOT (unary operation)
1DTT |      |   8   | Shift or Rotate, see section below (unary operation)

### Shift and Rotate

All shifts can be performed left or right, as designated by the `D` bit of the instruction.
If `D` is a `1`, the shift is to the left, all bits will move to a higher value, if `D` is `0`, it's a right shift, moving bits to lower values.
There are then four types of shift that can be performed designated by the final two bits of the ALU instruction.
The name should be appended with an L or R for the direction of the shift, left or right respectively.
For all shift operations, the bit shifted out is set into the Carry flag.

TT | Name | Description
---|------|------------
00 | LSF  | Logical shift - a zero is inserted
01 | ASF  | Arithmetic shift - a zero is inserted for left shift, bit-7 (MSB) is inserted for right shift
10 | RTC  | Rotate with carry - the Carry flag is inserted (Carry flag value before it is updated is used)
11 | RTW  | Rotate without carry - the bit shifted out is inserted

An example of a Arithmetic shift right;
`AXXX XXXB` would become `AAXX XXXX`, with `B` copied to the Carry bit.

**NB:** An 'Arithmetic shift left' is the same as performing a 'Logical shift left', they _can_ be used interchangeably, but 'Arithmetic shift left' should be avoided.

## STCK - Stack Manipulation

When dealing with the stack, a pair of registers will be moved to or from 'the stack' and the SP updated to reflect the changed address.
The registers A and B are paired, as are the registers C and D.
Effectively, the stack works on 16-bit values, but due to the 8-bit data bus it requires two transfers, though this is handled via the hardware/microcode.

The Stack manipulation operations are of pattern `1111 10DR`.
The `D` bit indicates the direction;
`0` for PUSH and `1` for POP.
The `R` bit indicates the register pair;
`0` for A & B and `1` for C & D.

When PUSHing, B/D will go to the address one less than the current SP, whilst A/C will go to address two less than the SP.
After PUSHing, the SP will have been decremented by two, with the SP containing the address of A/C (now in memory).

When POPing, the same respective pairs of memory locations will be read to the same pair of registers, and the SP increased by two.

Care must be taken, especially when POPing the stack, as there is no under/overflow protection or detection, just like with the PC incrementing during instruction execution.
In fact, by design, POPing the final value from the stack will result in an overflow bringing the SP back to `0x0000`.

In terms of pseudo-code a PUSH followed by a POP can view as the following microcode, where SP is a pointer to the memory address:

```CPP
// PUSH
SP -= 1
*SP = B
SP -= 1
*SP = A

// POP
A = *SP
SP += 1
B = *SP
SP += 1
```

**NB:** I Think I might update this to allow pushing/popping the PC, this would make it very easy (hardware wise) to handle calling and returning functions

## JUMP

This Instruction takes a three bit operand indicating under what condition the jump should be performed.
If the condition is met, the value of ADR is loaded into the PC.
If the condition is not met, no further special action is taken;
the PC would have already been incremented as part of loading the instruction.

**NB:** The value of ADR must have been set with the desired target location prior to the JUMP instruction being performed.

This table shows what combination of bits to the JUMP instruction check what flags and in what combination.

FFF | Name | Description
----|------|-------------
000 | JMPZ | Zero flag
001 | JMPP | Parity flag
010 | JMPG | NOT Zero AND Greater than flag (i.e. greater than)
011 | JMPC | Carry flag
100 | JMZG | Zero OR Greater than flags
101 | JMZL | Zero OR NOT Greater than flag
110 | JMPL | NOT Zero AND NOT Greater than flag (i.e. less than)
111 | JUMP | Unconditional Jump (always jumps)

## MADR - ADR Register Manipulation

The ADR register is a 16-bit register, it's value can be set/read to the general purpose registers.
The registers A and B are paired, as are the registers C and D.
Although still two distinct bytes, the A and C registers should be considered the more significant byte whilst B and D registers the lesser.

These ADR manipulation operations are of pattern `0000 10DR`.
The `D` bit indicates the direction;
`0` for write-to and `1` for read-from the ADR register.
The `R` bit indicates the register pair;
`0` for A & B and `1` for C & D.
