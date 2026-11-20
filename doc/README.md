# masc manual

masc is the assembler of the mach toolchain. It reads assembly source into
relocatable objects, prints objects back as source, checks functions for
secret-dependent timing, and gives compilers an instruction-level API. The
`masc` program is a command line over the library.

Every example in these pages was assembled, and where it applies
disassembled, with the `masc` program of version 0.1.0.

## Pages

- [Command line](command-line.md): every command, its options, what it reads
  and writes, and its exit codes.
- [Assembly source](source.md): the rules every instruction set shares:
  statements, comments, labels, names, numbers, expressions, directives,
  sections, symbols and relocations.
- [x86-64 and 32-bit x86](x86.md)
- [AArch64](aarch64.md)
- [RISC-V](riscv.md)
- [WebAssembly text](wasm.md)
- [SPIR-V text](spirv.md)
- [Disassembly and listings](disassembly.md): what `disassemble` and
  `listing` print, and the round trip each holds to.
- [Extensions and feature selection](extensions.md): how each set names its
  extensions and levels, and how a target, a flag or a directive chooses
  them.
- [Inline assembly](inline.md): the `inline` command and the block syntax a
  front end hands masc.
- [Constant-time walk](walk.md): the `walk` command and the effects data
  behind it.
- [Library](library.md): a map of the public modules a compiler builds
  against.

## Instruction sets at a glance

`masc isas` lists the registered sets:

```
riscv64 riscv32 riscv64e riscv32e
  syntaxes: riscv
  levels:   rv32i rv64i rv64gc rva20u64 rva20s64 rva22u64 rva22s64 rva23u64 rva23s64 rvb23u64 rvb23s64
x86 x86_64
  syntaxes: intel
  levels:   x86-64 x86-64-v2 x86-64-v3 x86-64-v4
wasm32 wasm64
  syntaxes: wat
  levels:   wasm1 wasm2 wasm3
spirv
  syntaxes: spirv
  levels:
aarch64 arm64e arm64ec
  syntaxes: aarch64
  levels:   armv8-a armv8.1-a armv8.2-a armv8.3-a armv8.4-a armv8.5-a armv8.6-a armv8.7-a armv8.8-a armv8.9-a armv9-a armv9.1-a armv9.2-a armv9.3-a armv9.4-a armv9.5-a armv9.6-a armv9.7-a armv9.8-a
```

The first line of each entry names the architectures the set serves, then
its one syntax and the levels its catalog names.
