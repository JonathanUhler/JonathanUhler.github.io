# Design of the `misa-as` Preprocessor

A separate utility that converts: String (text file) -> String (text file). For simplicity, isolated
from the assembler (which will only operate on preprocessed files).

For one given file, the preprocessing goes as follows:

- Parse the file as these semantic units:

  1) File inclusion
  2) Macro definition
  3) Plaintext

- Extract all the file inclusions. (Attempt to) read the files and copy-paste their content
  verbatim into where the inclusion directive was among the macros and plaintext. Run the
  parser recursively on included files before copying their contents
- Once all inclusions are recursively resolved and the preprocessor is back to a single file,
  extract all the macro definitions
- Search through the remaining plaintext components for macro keywords. If any are found, replace
  them verbatim with the macro values
- Stitch all the processed plaintext components back together in order (excluding the macro
  definition directives)

Really, this will probably require a two-pass implementation:

```
resolveInclusions content = do
  inclusions = parseForInclusions content
  for inclusion in inclusions do
    included_content = resolveInclusions (readFile inclusion)
    content = includeInclusion included_content
  return content

resolveMacros content = do
  macros = parseForMacros content
  for key, value in macros do
    content = replaceAll key value content
  return content

preprocess content = do
  content = resolveInclusions content
  content = resolveMacros content
  return content
```

## Syntax

To separate the preprocessor (handled externally, not supported by the assembler) from assembler
directives (explicitly handled by the assembler), preprocessor directives will follow the C
convention of using `#`.

The preprocessor will support:

- `#macro KEY VALUE #endm` for macro definitions. KEY must be an identifier, and VALUE can be
  literally anything
- `#include FILE` for external file inclusions

## Other Considerations

- The assembler CLI should be updated to follow the convention of running `.S` files through the
  preprocessor, and `.s` files only through the assembler (skipping preprocessing)
- The assembler CLI should support a flag to stop after the preprocessor (effectively just running
  as a wrapper for the preprocessor tool)
