# MISA Sim Design

The goal is to create a functional simulator to debug and run code, and not to do anything related
to hardware design or development.

The functional simulator won't implement any fancy hardware concepts like caching or pipelining. It
should execute a fetch-decode-execute loop once per call to some `step` function.

The main abstraction will be `Simulator`, which has the following interface:

- `reset`: Equivalent to applying power and/or toggling the reset signal in hardware. This function
  should perform the initialization required by the ISA on reset.
- `step`: Executes a single instruction at the program counter, which represents the smallest
  atomic unit of work defined in the ISA manual. `step` should fetch the 16-bit instruction word
  from memory at the PC value, decode the instruction, execute the instruction, and increment the PC
  (or branch, etc)

The `Simulator` class will have the following state that is modified on each simulation step:

- `pc`: The program counter, which is an absolute 16-bit address
- `reg`: The array of general purpose registers defined by the ABI
- `csr`: The array of special registers defined by the ABI
- `mem`: The memory attached to the processor
