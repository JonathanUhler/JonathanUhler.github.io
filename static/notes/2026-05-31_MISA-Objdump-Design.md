# MISA Objdump Design

The goal is to create a basic disassembler for use in the gdb-like interface of the functional
simulator. The disassembler should be able to disasseble MISA_LF object files, and also flat
binaries (when provided with some help like start address/offset/symbol file).

The main challenge will be dealing with the `.space` directive and (for flat binaries) the linker
choosing to insert space between sections. There might be an argument for making `.space` directives
add to a "SpaceTable" in object files, which are only unpacking during linking.

## Space Table Concept

If this were added, `.space` directives (and maybe `.align` in the future -- things that add an
amount of 0x00 bytes that is undefined until linking) would be added to a structure like:

```haskell
type SpaceTable = [Space]

-- Space <relative address in Sec's code section> <number of blank bytes in the space>
-- The space will be inserted before whatever byte is currently at the address
data Space = Space Word16 Word16

-- Updated to add the space table
data Sec = Sec ... SpaceTable
```

The space table would be encoded like the symbol/relocation tables, by having a count (number of
entries in the space table) followed by that many entries (which are just the two `Word16` values).
This also has the benefit of reducing object file sizes with large `.space` directives.

In the linker, the implementation would probably be best by having a new pass that does the
following on every `Sec` of every `BinaryObject`:

- Rewrite the code portion of every `Sec` to include the entries in the space table
- For each symbol or relocation in the `Sec`, increase its relative address by the space added
  from all `Space` entries if the sym/reloc address >= the space address
  
Then, the rest of the linker can remain unchanged if it's passed these rewritten object files. An
example of the '>=' logic:

```text
// Some instruction
foo:          --> generates Sym "foo" 0x0000
0x0000  0xAA
0x0001  0xBB

.space n      --> generates Space 0x0002 n

// Some other instruction
bar:          --> generates Sym "bar" 0x0002
0x0002  0xCC
0x0003  0xDD

// Only "bar" needs to be rewritten since its address is == the space address
// The new "bar" address becomes original address + space count = 0x0002 + n
```

## Object File Decoder

With space tables implemented, the objdump implementation should be able to accurately reverse
object file code without having to guess as much. It may still run into issues with other directives
that add arbitrary (non-space) data like `.array`, however the GNU objdump doesn't deal with this
either.

Given an object file, the decoder will do the following:

- Assume all the data in the object file code section is actual code, and start decoding at the
  very start of the code section.
- Read two bytes and attempt to interpret an opcode, and then the operands for that instruction.
- Continue until the end of the code section.
- If any invalid instruction is found, output some sentinel like "(bad)" because the object file
  is either bad, or there is some non-instruction data that caused an alignment issue for the
  decoder.
  
This process should be able to take an optional start address (zero by default), which the user
can pass through the CLI.

By default, the decoder will only attempt to decode sections called "text". Non-text sections will
be output/printed as hexdumps. Optionally, the decoder can be made to attempt to decode all sections
as code.
