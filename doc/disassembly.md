# Disassembly and listings

masc prints source from two places. `disassemble` reads an object file's
bytes. `listing` reads what the builder was asked for while it assembled a
source. Both print text that `assemble` reads back.

## disassemble

`disassemble` reads the object through mink's registered formats, its format
sniffed from its magic, so any format mink converts to its neutral object
disassembles alike. It prints, in order:

1. the symbols' bindings, visibilities, kinds and sizes, and the equated
   names, before any section
2. each section under the directive that enters it, with its symbols as
   labels at their offsets
3. each code section decoded and printed in the set's syntax
4. every other section, and any bytes of a code section that decode as no
   instruction, as data directives

A relocation prints as the symbol of the operand it patches, under the
specifier the syntax spells its kind by, or as a datum, as `.quad sym`. A
relocation no statement takes is stated by a `.reloc` after the statement it
patches, by the format's name of its kind:

```
	.section .pdata, "dr", associative, .text
	.balign 4
	.reloc 0, IMAGE_REL_AMD64_ADDR32NB, .Ltmp0
	.byte 0x00, 0x00, 0x00, 0x00
```

A place a relocation names that has no symbol of its own, as a section
symbol, gets a label of the dialect's temporary stem, `.Ltmp0` and on.

### The round trip

A statement is printed only when it reads back to the same bytes: its text
is parsed again as the assembler parses it, given the shortest form the
assembler would give it, and encoded, and the bytes and relocations must be
the ones the object holds. A statement that does not read back is printed as
data, never as a guessed instruction.

```
$ masc assemble b.s -t linux-x86_64 -o b.o
$ masc disassemble b.o -t linux-x86_64 -o b2.s
$ masc assemble b2.s -t linux-x86_64 -o b2.o
```

`b.o` and `b2.o` hold the same section bytes and the same relocations. The
order of the symbols in the symbol table may differ.

The code is decoded under the target's selection and state, which the flags
give, not under what the source said. A source that widened its selection
with a directive needs the same selection to disassemble as instructions:

```
    .isa x86-64-v3
    .globl sum8
sum8:
    vaddps ymm0, ymm0, ymm1
    ret
```

```
$ masc disassemble avxd.o -t linux-x86_64
	.globl sum8
	.section .text, "ax", @progbits
sum8:
	.byte 0xc5
	cld
	pop rax
	.byte 0xc1
	ret
$ masc disassemble avxd.o -t linux-x86_64 --isa x86-64-v3
	.globl sum8
	.section .text, "ax", @progbits
sum8:
	vaddps ymm0, ymm0, ymm1
	ret
```

Both texts assemble back to the same bytes. The first reads the vex prefix
under a selection without avx, where no row decodes it.

Where the object's mapping symbols name the selection the code was written
under, as RISC-V's do, the disassembler reads the code under it and restates
it with the dialect's directive, as `.option arch, rv32i2p1_..._zbb1p0` (see
[RISC-V](riscv.md)). A statement that reads back only under a narrower scope
is written after the directives that move to it, as RISC-V's `.option push`,
`.option norvc` and `.option pop`.

The x86 code width is not in the object, so code assembled after `.code32`
inside an x86_64 object decodes at 64 bits, and the byte `40` of a 32-bit
`inc eax` prints as the `{rex}` marker of the instruction after it. The
bytes still read back the same.

### What the text does not state

The traits an object states that the caller gives the assembler, as the
selection or the SPIR-V version, are noted on stderr so none is left out
silently:

```
note[disasm.unwritten] the text does not state trait riscv-arch rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0, which the selection --isa names gives the assembler again (a.o)
```

`-q` drops these notes. What the text cannot state at all is refused with
`disasm.refused`, naming the section and offset, with nothing printed: a
group that is no comdat group, a link-order owner no name reads back to,
note records, index entries, a version record and a relocation that
subtracts a symbol.

Line records are restated as `.file` and `.loc` lines. Call frame records
are not: an object assembled with `.cfi_*` directives disassembles to text
without them, with no note, and that text assembles to an object with no
`.eh_frame`. Windows unwind data prints as the `.xdata` and `.pdata`
sections' bytes and their `.reloc` lines.

A later section of a name already entered is opened apart by the dialect's
unique number on ELF, as `.section .text.hot, "ax", @progbits, unique, 1`,
or by its leader on COFF, and is refused under a dialect that keeps no two
sections of one name apart.

### Aliases

Instructions print by their preferred alias unless `--no-aliases` is given:

```
$ masc disassemble a.o -t linux-aarch64 --no-aliases
...
sum:
	movz x2, #0, lsl #0
...
	orr x0, xzr, x2, lsl #0
```

### Modules

A format whose text states one module, as wasm and SPIR-V, has the object
written as that module. A SPIR-V module prints as the statements of its one
instruction stream, and a wasm module as its fields. Either is held to the
bytes the format writes, so the printed text assembles to a file of the same
bytes.

## listing

`listing` assembles the source and prints the builder's record of what it
was asked for: each section's statements as written, instructions by the row
they were written in, symbols as symbols, data, fills, alignments and
differences of labels. It writes no object. Every encoding choice the
builder made is pinned in the text, and a statement written under another
view than the object's is written under the directives that make it.

```
$ masc listing d.s -t linux-x86_64
	.intel_syntax noprefix
	.globl "odd name"
	.section .text, "ax", @progbits
"odd name":
	mov rax, 1
	ret
.Lhidden:
	.balign 16
	.code32
	inc eax
	.code64
	jmp .Lhidden
```

The same object disassembled:

```
$ masc disassemble d.o -t linux-x86_64
	.globl "odd name"
	.section .text, "ax", @progbits
	.balign 16
"odd name":
	mov rax, 1
	ret
.Ltmp0:
	{disp32} nop dword ptr [rax + rax]
	{rex} jmp .Ltmp0
```

The listing keeps what only the source knew: the alignment as an alignment,
the `.code32` width, the private label's name and the call frame directives.
Disassembly keeps only what the bytes and the object hold. A listing also
writes a symbol's size as a difference of labels, as `.size add, .Ltmp0 -
add`, and a symbol's local binding, as `.local f`.

On RISC-V a listing shows the compressed form the builder chose, as
`c.add a0, a1`, where disassembly prints the preferred alias, `add a0, a0,
a1`.

A wasm source is refused with `listing.module`, as its text states its
module whole.
