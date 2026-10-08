# masc

<p>
  <a href="https://github.com/briar-systems/masc/actions/workflows/ci.yml"><img src="https://github.com/briar-systems/masc/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/briar-systems/masc?color=FF00FF&labelColor=000000" alt="License"></a>
</p>

**The assembler of the mach toolchain, written in [Mach](https://github.com/briar-systems/mach).**

masc owns instruction sets: their registers, encodings, extensions, assembly
syntax and the effects of each instruction. It reads assembly text and writes
relocatable objects, disassembles any object back into text that assembles to
the same bytes, and gives compilers an instruction-level API so code
generation and inline assembly never go through text.

It covers x86-64 and 32-bit x86, AArch64, RISC-V, and the typed instruction
streams of SPIR-V and WebAssembly, including the WebAssembly text format in
full. Each set is implemented to its specification, with catalogs generated
from machine-readable sources wherever one exists.

masc is a library first. The `masc` program is a command line over the same
API.

> masc is early work. The design is settled but most of what is described
> here is not built yet.


## Place in the toolchain

The mach toolchain is four projects, each depending only on the layers below it.

```
mach   language front end
 └ mirl   IR and code generation
    └ masc   assembler
       └ mink   linker and object formats
```

masc depends on [mink](https://github.com/briar-systems/mink) for
architectures and object formats, and on the Mach standard library. mirl and
mach depend on masc.


## Architecture

- **The core.** One row schema describes an instruction in every set: its
  operands, its encoding as fields, the extensions it requires, its effects
  and its constant-time class. Encoding and decoding walk the same fields in
  two directions.
- **Instruction sets.** One module per set, each with its register file, its
  extension catalog and its generated catalog of rows. Adding a set is one
  registration and rows.
- **Syntax, assembly and disassembly.** Each set's established assembly
  syntax with the GNU assembler's directives, read into objects and printed
  back from them.
- **The builder.** The instruction-level API a compiler emits through, one
  instruction at a time, with labels, relaxation, relocations and a listing of
  what was encoded.
- **Inline assembly.** Parsing and effects for a front end at semantic
  analysis, and encoding with real registers after register allocation.
- **Walks.** Checks over encoded code driven by instruction effects, such as
  secret-dependent timing.
- **Generators and harness.** The `gen` program, its own project under
  `gen/`, builds catalogs from the sources vendored under `gen/vendor`, and the
  `harness` program checks masc against external tools. Both run locally and
  are never linked into the library.


## Building

masc is built with the current release of the [Mach compiler](https://github.com/briar-systems/mach/releases).

```bash
git clone https://github.com/briar-systems/masc.git
cd masc
mach dep pull .
mach build .
mach test .
```


## License

[MIT](LICENSE)
