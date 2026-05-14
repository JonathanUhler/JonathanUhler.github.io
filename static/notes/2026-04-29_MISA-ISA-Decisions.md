# MISA ISA Decisions

This notesheet details the decision-making process for the core instruction set of the
[MISA](https://github.com/JonathanUhler/MISA) architecture. This is a retrospective written
after deciding the list of base instructions.

# Number of Instructions

While [single-instruction machines](https://esolangs.org/wiki/OISC) exist and are Turing-complete,
the goal of MISA is to provide a minimal ISA that actually has some quality of life for the
programmer or compiler developer.

The number of instructions was motivated by the following:

I was committed to a load-store architecture, which would mean the maximum instruction would take
three operand registers to perform the binary operation `z = f(x, y)`.

Since the native word size is 8 bits, I wanted a "set register" instruction that could take an
entire 8-bit immediate value...which would require the instruction word to be 16-bits (since 8 are
taken by just the immediate, and loading halfwords isn't something I wanted to do).

A 32-deep register file would take 15 bits, leaving 1 for the opcode, which obviously wouldn't work.
A 8-deep register file wouldn't give that much to work with, especially if 16-bit values (like
addresses) needed to be stored in registers. So 16-registers would require 12 bits for operand
encoding, leaving 4 bits for opcode, or 16 possible instructions.
  
# Instruction Choices

MISA needed well-defined functionality for:

- Arithmetic operators
- Bitwise operators
- Use of constants
- Memory access
- Control flow

From this requirement I began including instructions with the following reasoning:

- ADD: This is the most logical arithmetic instruction to include. Multiplication is more expensive
  in hardware, and can be simulated with addition in a loop. Subtraction can be written as addition
  (but more on that later). There could be an argument for an increment instruction (ADD with one
  of the operands fixed as 1), but this would make addition as complex in software as multiplication
  is with addition, which I decided was too much of a burden on the programmer and performance.
- ADC: I wanted to have somewhat convenient support for 16-bit arithmetic, which would have to
  require ~2 instructions per operation, but wouldn't be terrible to write in assembly. Assuming
  the ALU carry bit is easily accessibly, a 16-bit add could be done with just the 8-bit ADD
  instruction, but would require double the instructions per wide addition. So, that's why there's
  ADC, since 16-bit support is an intentional feature rather than something that is just "possible".
- AND, OR, and XOR: Like with arithmetic, I wanted bitwise operations to be relatively convenient,
  which ruled out the possibility of trying to do something fancy with one instruction and universal
  gate conversions. AND and OR seemed like obvious choices because they are very frequently used
  logical connectives. I chose XOR because, although it isn't as common as AND/OR, it's still
  pretty common and gives bitwise negation with a short-ish 2 instruction sequence (load 0xFF, XOR).
  I didn't include negation as a dedicated instruction because it's not as useful as XOR in that
  regard, even if it's used more frequently. Also, using 4/16 instructions for basic bitwise
  operators was more than I was willing to do.
- RRC: For bitwise operators, I wanted to support shifting. Left-shift by 1 is the same as
  multiplying by 2, which is the same as adding a number to itself. Since that is already supported
  with ADD, I decided I didn't need a dedicated shift-left instruction. For left rotation, the bit
  manipulation needs to be done by hand with masking, but I don't remember the last time I've needed
  to rotate left in academic, personal, or professional work, so no need for that instruction. That
  leaves right shifting and rotating, which is hard to do because of the required division (if
  encoding it the same was as x << 1 <--> x + x). So, that motivated having a right-shift
  instruction. Since I have a carry bit for the ALU anyway, I realized I could support right
  rotations pretty easily by rotating the 9-bit value {regsiter, carry}. If carry is cleared before
  the rotation, RRC acts like a shift. Using the carry bit also allows easier 16-bit shifting.
- SET: I didn't intend on supporting immediate operands for any instruction. Having separate
  "immediate" and "register" versions would use too many opcodes, and register-based operations
  are more common in the RISC ISAs I've used in the past, so they were chosen over immediate-based
  instructions. But being able to set register values from constants is very useful, hence SET as
  the only core instruction supporting immediate values.
- LD and ST: These fulfill my minimum criteria for memory access. They simply allow reading and
  writing single bytes/words. The main design decision here was the use of absolute addressing, but
  that is to be justified in another writeup.
- RSR and WSR: I decided somewhat early on that I would include a separate set of special-purpose
  registers for processor control. The first one was FLAGS for the ALU. When I was thinking about
  the programmer ABI, I decided that it would be useful to have the stack and return addresses
  be in dedicated registers, rather than consuming 1/4 of the GP register file. These instructions
  are inspired by the Xtensa ISA, which manages processor state through special registers. For just
  two opcodes, there is a (slightly less convenient) ability to access an arbitrary number of
  additional registers for decidated data.
- JAL and JMP: For control flow, I needed some sort of conditional branch ability. The logic for
  calculating all of the inequalities (==, !=, <, <=, etc) is not that expensive in terms of gates
  with the ALU flags I have access to. Thus, both JAL and JMP support all the conditions in one
  instruction. Since the return address is a special register, JMP exists as a way to *not* link
  the return address.
