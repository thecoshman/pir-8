Unless stated otherwise, bits are always represented from MSB to LSB (reading left to right) and multi-bytes sequences are big-endian.
So, a jump instruction followed by a two byte address would have the following sequence of bytes "jump", "high address byte", "low address byte".

# Registers

## Special Purpose Registers

There are some special purpose registers that you cannot directly read/write from, these are used by the CPU for it's internal state. 
There are three 16 bit registers for holding significant memory addresses and a single 8 bit register.

Name            | Short | Bits | Description
----------------|-------|------|------------
Program Counter | PC    |  16  | Address of the next instruction to be fetched
Stack Pointer   | SP    |  16  | Current address of the stack (detailed later)
Memory Address  | ADR   |  16  | Current address of RAM being accessed
Instruction     | INS   |   8  | Instruction currently being executed

## General Purpose Registers

There are 8 General Purpose registers.
Each has an internal address for use within the CPU, instructions like 'MOVE' and 'LOAD' can use these addresses.
The first four registers have some special functionality as described;
the second four have no special functionality, but can be used with the stack.

Address | Letter | Description
--------|--------|------------
000     | F      | Flag register (can also be used to get a zero value)
001     | S      | Output of the ALU - ALU operations will overwrite any value stored
010     | X      | Input to ALU (Only input for unary operations)
011     | Y      | Second input for ALU
100     | A      | 
101     | B      | 
110     | C      | 
111     | D      | 

## Flag register

The flag register can be be read and wrote to as a general purpose register, though note what instructions can overwrite those flags.
ALU and Compare instructions can effect the value of the flags.
Below is a description of what each bit in the flag register denote.
Not all of the bits have a specified role (yet), though the CLRF operation will still clear them.

Bit | Letter | Description
----|--------|------------
0   | Z      | Zero flag
1   | C      | Carry flag
2   | P      | Parity (even number of high bits)
3   | E      | Equals flag
4   | G      | Greater than (X greater than Y, treating both as unsigned values)
5   |        | 
6   |        | 
7   |        | 

# Instructions

All instructions will increase the program counter (PC) by one, unless otherwise stated.
The PC is incremented as the instruction is loaded from RAM.

The 'Bit Mask' shows a mask which denotes an instruction or group of instructions.
The 'name' is for either a group or single instruction, assembly should favour these names.
'Count' is how many of the 256 possible instructions are used by that bit pattern; HALT for example is exactly one instruction, whilst MOVE is effectively 64 possible combinations.

Bit Mask  | Name | Count | Description
----------|------|-------|------------
0000 XXXX |      |    16 | Reserved
0001 0XXX | JUMP |     8 | Jump, see section below
0001 1AAA | LOAD |     8 | Load the the next byte into register `AAA` (PC will be incremented a second time)
0010 0AAA | LOAD |     8 | Load value in address indicated by the next two bytes into register `AAA` (PC will be incremented two more times)
0010 1AAA | SAVE |     8 | Store value in register `AAA` in address indicated by the next two bytes (PC will be incremented two more times)
0011 XXXX | ALU  |    16 | ALU based operations, see section below
01AA ABBB | MOVE |    64 | Move a value from register `AAA` to register `BBB`
10XX XXXX |      |    64 | Reserved
110X XXXX |      |    32 | Reserved
1110 XXXX |      |    16 | Reserved
1111 0AAA | COMP |     8 | Compare register S with register `AAA`, see section below
1111 10XX | STCK |     4 | Stack manipulation, see section below
1111 110X |      |     2 | Reserved
1111 1110 | CLRF |     1 | Clear the 'F' register, by setting it to `0000 0000`
1111 1111 | HALT |     1 | Stop the CPU from doing any more execution 

## COMP - Compare

The compare instruction will compare the S register with a selected register.
It will set the Zero and Parity flag based on the value of the S register;
Zero flag if all the bits are zero, Parity if the number of one bits is even.
Will set the Equal flag if the two registers have the same bit pattern;
Note that the equality check does not have signed/unsigned handling.

NB: This might change to instead compare just the X and Y register.

## ALU Instructions

Any CPU instruction of the pattern `0011 BFFF` will invoke some function of the ALU.
The `B` bit indicates if we are doing arithmetic operations (B is 0) or bitwise operations (B is 1).
The final three bits `FFF` are the actual operation being performed. 
The registers X and Y are used as inputs to the ALU (only X for unary operations), and the S register is used stores the result.

### Flags

Some operations will also update the F register as noted.
All will set the ZERO flag if the output (register S) is `0000 0000`.
All will set the Parity flag if the number of high bits are even.

### Arithmetic Operations

NB: Currently thinking I might re-order these into "unary" and "binary" operations. The 'shift' functionality might also be expanded to provide more ways of shifting and rotating

These are all of pattern `0011 0FFF`.

FFF | Name | Count | Description
----|------|-------|------------
0XX |      |   4   | Reserved
100 | ADD  |   1   | Addition of register X and register Y
101 | SUB  |   1   | Subtraction of register Y from register X (X-Y)
11X |      |   2   | Reserved

### Bitwise Operations

These are all of pattern `0011 1FFF`.
The NOT and 'shift' operations are unary operations, taking the input from just register X; OR, XOR and AND use both X and Y registers as input

FFF | Name | Count | Description
----|------|-------|------------
000 | NOT  |   1   | Bitwise NOT
001 | OR   |   1   | Bitwise OR
010 | XOR  |   1   | Bitwise XOR
011 | AND  |   1   | Bitwise AND
10A | SHFL |   2   | Shift left (to higher bits) of register X, inserting 'A' into the LSB (ie 1101 1101 -> 1011 101A), the bit shifted out is stored into the carry bit of the F register
11A | SHFR |   2   | Shift right (to lower bits) of register X, inserting 'A' into the MSB (ie 1101 1101 -> A110 1110), the bit shifted out is stored into the carry bit of the F register

## Stack Manipulation

When dealing with the stack, a pair of registers will be moved either to or from 'the stack' and the SP updated to reflect the changed address.
The registers A and B are paired, as are the registers C and D.
Effectively, the stack works on 16 bit values, but due to the 8 bit data bus it requires two transfers, though this is handled via the hardware/microcode.
Although still two distinct bytes, the A and C registers should be considered the more significant byte whilst B and D registers the lesser; 
the more significant byte will be stored at the lower address in the stack, the pair of registers are big-endian.

The Stack manipulation operations are of pattern `1111 10DR`.
The D bit indicates the direction; 0 for PUSH and 1 for POP.
The R bit indicates the register pair; 0 for A & B and 1 for C & D.

When PUSHing B or D will go to the address of the SP, whilst A or C will go to address one less than the SP.
After PUSHing, the SP will have been decremented by two.

When POPing, the same respective pairs of memory locations will be read from the same pair of registers, and the SP increased by two.

## Jump

The Instruction takes a three bit operand indicating under what condition the jump should be performed.
If condition is met, the next two bytes after this Jump instruction are loaded into the PC.
If the condition is not met, the PC is incremented by two.

This table shows what combination of bits to the JUMP instruction check what flags and in what combination

FFF | Description
----|------------
000 | Zero flag
001 | Parity flag
010 | Greater than flag
011 | Carry flag
100 | Zero OR Greater than flags
101 | Zero OR NOT Greater than flag
110 | Unconditional Jump PUSH return address (always jumps, will use the S & F registers to hold the target address - this will need be clear manually)
111 | Unconditional Jump (always jumps)
