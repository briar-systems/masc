# AArch64

One set serves `aarch64`, `arm64e` and `arm64ec`. Its one syntax is
`aarch64`, the vendor dialect, its templates generated from Arm's instruction
model.

| fact               | AArch64                                              |
|--------------------|------------------------------------------------------|
| line comment       | `//`                                                 |
| separator          | `;`                                                  |
| `.align n`         | 2^n bytes                                            |
| integer data       | `.byte` 1, `.hword` `.short` `.2byte` 2, `.word` `.long` `.int` `.4byte` 4, `.xword` `.dword` `.quad` `.8byte` 8, `.secrel32` 4 (a section-relative datum) |
| symbol kind prefix | `@` or `%`, as `%function`                           |

A `#` starts no comment here, so a file whose first line is a `#` comment is
refused on AArch64 targets.

## Directives of the set

| directive                | meaning                                                  |
|--------------------------|----------------------------------------------------------|
| `.arch level+mods`       | restate the selection by an architecture level and its modifiers |
| `.arch_extension name`   | add an extension, or with `no` before its name, as `nocrc`, remove it with every extension that implies it |
| `.inst word, ...`        | instructions by their 32-bit encodings                   |
| `.tlsdesccall sym`       | mark the `blr` after it with `R_AARCH64_TLSDESC_CALL`    |

Nothing states what an `.inst` encoding does, so the walk and the effects
treat it as one that may read and write every register and memory and jump
anywhere.

```
    .text
    .arch armv8.1-a
    ldadd x0, x1, [x2]
    .arch_extension crc
    crc32b w0, w1, w2
    .inst 0xd503201f
```

```
$ masc assemble ext.s -t linux-aarch64 -o ext.o -v
ext.s: 1 sections, 1 symbols, 0 relocations, written to ext.o as elf
$ masc disassemble ext.o -t linux-aarch64 --isa armv8.1-a+crc
	.section .text, "ax", @progbits
	.balign 4
	ldadd x0, x1, [x2]
	crc32b w0, w1, w2
	nop
```

## Operands

Registers are written by their own names, `x29` rather than `fp`, and read by
those and their aliases. Mnemonics and names read without regard to case. An
immediate takes an optional `#`.

- A register list reads comma separated or as a range, as `{ z0.b - z3.b }`,
  and a range may wrap past the class's last register. Lists are written
  comma separated, and as a range when the class is scalable and three or
  more members run up without wrapping.
- A float immediate reads as a decimal number, as `#-1.25` or `#2`, and
  writes with eight places, as `#-1.25000000`.
- A system register reads by its name or by its encoding as `s3_0_c15_c2_0`.
- An operand named by name or number, as `dsb ish` or `dsb #12`, reads either
  way and writes by name where a name exists.

## Relocation specifiers

| formats       | specifiers |
|---------------|------------|
| ELF and COFF  | `:lo12:`, `:abs_g0_nc:`, `:got:`, `:got_lo12:`, `:gottprel:`, `:gottprel_lo12:`, `:tlsdesc:`, `:tlsdesc_lo12:`, `:tprel_g0:` to `:tprel_g2:`, `:tprel_g0_nc:`, `:tprel_g1_nc:`, `:tprel_hi12:`, `:tprel_lo12:`, `:tprel_lo12_nc:`, `:dtprel_g0:` to `:dtprel_g2:`, `:dtprel_g0_nc:`, `:dtprel_g1_nc:`, `:dtprel_hi12:`, `:dtprel_lo12:`, `:dtprel_lo12_nc:` |
| ELF           | `@DTPREL` on a 64-bit datum |
| COFF          | `:secrel_hi12:` on an add, `:secrel_lo12:` on an add, load or store, `@SECREL32` on a 32-bit datum |
| Mach-O        | `@PAGE`, `@PAGEOFF`, `@GOTPAGE`, `@GOTPAGEOFF`, `@TLVPPAGE`, `@TLVPPAGEOFF` |

The colon specifiers go before the symbol, as `:lo12:table`, and the `@`
specifiers after it. Mach-O's dialect reads no colon specifier.

## Example

```
// aarch64: comments run from //
    .text
    .globl sum
    .type sum, %function
    .p2align 2
sum:
    mov x2, #0
1:  cbz x1, 2f
    ldr x3, [x0], #8
    add x2, x2, x3
    sub x1, x1, #1
    b 1b
2:  mov x0, x2
    adrp x9, table
    add x9, x9, :lo12:table
    bl helper
    ret
    .size sum, . - sum

    .data
    .p2align 3
table:
    .xword sum
    .word 0x12345678
    .hword 7
```

```
$ masc assemble a.s -t linux-aarch64 -o a.o -v
a.s: 2 sections, 5 symbols, 4 relocations, written to a.o as elf
```

The text holds `R_AARCH64_ADR_PREL_PG_HI21` and `R_AARCH64_ADD_ABS_LO12_NC`
against `table` and `R_AARCH64_CALL26` against `helper`, and the data
`R_AARCH64_ABS64` against `sum`. Disassembly prints instructions by their
preferred aliases:

```
$ masc disassemble a.o -t linux-aarch64
	.globl sum
	.type sum, @function
	.size sum, 44
	.globl helper
	.section .text, "ax", @progbits
	.balign 4
sum:
	mov x2, #0
.Ltmp0:
	cbz x1, #.Ltmp1
	ldr x3, [x0], #8
	add x2, x2, x3
	sub x1, x1, #1
	b #.Ltmp0
.Ltmp1:
	mov x0, x2
.Ltmp2:
	adrp x9, #table
	add x9, x9, #:lo12:table
.Ltmp3:
	bl #helper
	ret
	.section .data, "aw", @progbits
	.balign 8
table:
	.xword sum
	.byte 0x78, 0x56, 0x34, 0x12, 0x07, 0x00
```

and with `--no-aliases` by their own forms, as `movz x2, #0, lsl #0` and
`orr x0, xzr, x2, lsl #0`.

The default selection of `linux-aarch64` is `armv8-a+fp+simd`. A level alone
admits only what the architecture makes mandatory, so `armv8-a` admits no
floating-point instruction. See [extensions](extensions.md).
