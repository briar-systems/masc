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
ext.s: 1 section, 1 symbol, 0 relocations, written to ext.o as elf
$ masc disassemble ext.o -t linux-aarch64 --isa armv8.1-a+crc
	.section .text, "ax", @progbits
	.balign 4
	ldadd x0, x1, [x2]
	crc32b w0, w1, w2
	nop
```

## Windows unwind codes

On a COFF target an AArch64 procedure states its unwind codes by these
`.seh_*` directives, between `.seh_proc` and `.seh_endproc` (see
[source](source.md#call-frames-unwind-codes-and-lines)). Each follows the
prologue instruction it describes. `n` is a constant number of bytes, an
offset is from the stack pointer, and a pre-indexed save moves the stack down
by `n` and saves at its new top. A pair is a register and the one after it,
general registers in their `x` form and floating point ones in their `d`
form.

| directive                     | what it records                                          |
|-------------------------------|----------------------------------------------------------|
| `.seh_stackalloc n`           | the stack moved down by `n` bytes                        |
| `.seh_save_r19r20_x n`        | `x19` and `x20` saved pre-indexed, `n` at most 248       |
| `.seh_save_fplr_x n`          | `x29` and `x30` saved pre-indexed                        |
| `.seh_save_fplr n`            | `x29` and `x30` saved at offset `n`                      |
| `.seh_save_lrpair xN, n`      | `xN` and `x30` saved at offset `n`                       |
| `.seh_save_regp_x xN, n`      | the general pair from `xN` saved pre-indexed             |
| `.seh_save_regp xN, n`        | the general pair from `xN` saved at offset `n`           |
| `.seh_save_fregp_x dN, n`     | the floating point pair from `dN` saved pre-indexed      |
| `.seh_save_fregp dN, n`       | the floating point pair from `dN` saved at offset `n`    |
| `.seh_save_reg_x xN, n`       | `xN` saved pre-indexed                                   |
| `.seh_save_reg xN, n`         | `xN` saved at offset `n`                                 |
| `.seh_save_freg_x dN, n`      | `dN` saved pre-indexed                                   |
| `.seh_save_freg dN, n`        | `dN` saved at offset `n`                                 |
| `.seh_save_any_reg_px r, n`   | `r` and the next register of its kind saved pre-indexed  |
| `.seh_save_any_reg_p r, n`    | `r` and the next register of its kind saved at offset `n` |
| `.seh_save_any_reg_x r, n`    | `r` saved pre-indexed                                    |
| `.seh_save_any_reg r, n`      | `r` saved at offset `n`                                  |
| `.seh_set_fp`                 | `x29` set to the stack pointer, the frame register       |
| `.seh_add_fp n`               | `x29` set to the stack pointer plus `n`, the frame register |
| `.seh_nop`                    | a prologue or epilogue instruction that states no unwind operation |
| `.seh_endprologue`            | the end of the prologue                                  |

The `save_any_reg` family takes any of `x0` to `x30` and any SIMD register in
its `d` or `q` form, which a `q` name saves 16 bytes wide. The other saves take
only the class and form their row names. Between `.seh_startepilogue` and
`.seh_endepilogue` a code follows the instruction that undoes the prologue
operation it names: a stack move moves the stack back, a save restores its
registers, a pre-indexed one moves the stack back past them, and a frame
register returns the canonical frame address to the stack pointer.

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
