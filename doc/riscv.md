# RISC-V

One set serves `riscv64`, `riscv32`, `riscv64e` and `riscv32e`. The width and
the base are state, not extensions: an isa string states both, `rv32` or
`rv64` and `i` or `e`. Its one syntax is `riscv`, the vendor dialect.

| fact               | RISC-V                                                |
|--------------------|-------------------------------------------------------|
| line comment       | `#`                                                   |
| separator          | `;`                                                   |
| `.align n`         | 2^n bytes                                             |
| integer data       | `.byte` 1, `.short` 2, `.word` and `.long` 4, `.quad` 8, `.dtprelword` 4 and `.dtpreldword` 8 (module-relative thread-local offsets) |
| symbol kind prefix | `@` or `%`                                            |

## Directives of the set

| directive                     | meaning                                              |
|-------------------------------|------------------------------------------------------|
| `.option rvc`                 | add `c` back to the selection                         |
| `.option norvc`               | remove `zca`, and with it `c` and every `zc*` that implies it, so the statements after it stay full length |
| `.option arch, spelling`      | restate the selection by an isa string, or by `+` and `-` edits of it |
| `.option push`, `.option pop` | save and restore the selection, narrowings, state, modes and syntax in force     |
| `.option pic`, `.option nopic` | the position mode, which decides what `la` loads    |
| `.option relax`, `.option norelax` | whether relocations carry `R_RISCV_RELAX`        |
| `.attribute tag, value`       | the psABI's object attributes, by tag name or number |
| `.insn format operands`       | an instruction by its encoding                        |

`.insn` takes the formats of the RISC-V assembly manual, as
`.insn r 0x33, 0, 0, a0, a1, a2`, and raw forms stating the whole encoding as
one value. Nothing states what such an encoding does, so the walk and the
effects treat it as one that may read and write every register and memory
and jump anywhere.

```
# rv32 with an extension stated in the source
    .option arch, +zbb
    .globl pop
pop:
    cpop a0, a0
    .insn r 0x33, 0, 0, a0, a1, a2
    ret
```

```
$ masc assemble b.s -t linux-riscv32 --abi ilp32 --isa rv32imac -o b2.o -vv
note[target] linux-riscv32, abi ilp32, isa rv32imac, format elf
b.s: 1 section, 2 symbols, 0 relocations, written to b2.o as elf
$ masc disassemble b2.o -t linux-riscv32 --abi ilp32 --isa rv32imac -q
	.globl pop
	.section .text, "ax", @progbits
	.balign 2
pop:
	.option arch, rv32i2p1_m2p0_a2p1_c2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zbb1p0
	cpop a0, a0
	add a0, a1, a2
	ret
	.option arch, rv32i2p1_m2p0_a2p1_c2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0
```

The object's mapping symbols carry the selection the code was written under,
so the disassembler restates `.option arch` around the code that needed
`zbb`. The `.insn` encoding reads back as the instruction it is.

## Operands

Registers are written by their ABI names, as `a0` and `ft1`, and read by
those or their `x`, `f` and `v` names. Mnemonics, names and specifiers read
without regard to case. A load or store address is written
`offset(register)`, as `lw a1, %lo(factor)(a1)`.

## Pseudo instructions and compression

`li` builds its constant from the shortest sequence of instructions the
selection admits. `call`, `tail`, `lla` and `la` are sequences of two
instructions under one relocation pair. `la` loads the address through the
GOT under `.option pic` or `--pic`, and directly otherwise. Respellings of
one instruction, as `ret`, `mv` and `nop`, are aliases.

When the selection holds `c` or `zca`, a base instruction is written in its
compressed form where one exists, so `add a0, a0, a1` assembles to
`c.add a0, a1`. A branch or jump to a label is compressed and grown back by
the layout when the label is out of reach. `.option norvc` keeps
instructions full length.

## Relocation specifiers

`%hi(sym)`, `%lo(sym)`, `%pcrel_hi(sym)`, `%pcrel_lo(label)`,
`%got_pcrel_hi(sym)`, `%tprel_hi(sym)`, `%tprel_lo(sym)`, `%tprel_add(sym)`,
`%tls_ie_pcrel_hi(sym)`, `%tls_gd_pcrel_hi(sym)`, `%tlsdesc_hi(sym)`,
`%tlsdesc_load_lo(sym)`, `%tlsdesc_add_lo(sym)`, `%tlsdesc_call(sym)` and
`%dtprel(sym)`, from the psABI's relocation operators.

## Example

```
# risc-v
    .text
    .globl scale
    .type scale, @function
    .p2align 1
scale:
    li t0, 1000
    mul a0, a0, t0
    lui a1, %hi(factor)
    lw a1, %lo(factor)(a1)
    add a0, a0, a1
    la a2, factor
    call helper
    ret
    .size scale, . - scale

    .option push
    .option norvc
    .globl full
full:
    addi a0, a0, 1
    ret
    .option pop

    .data
    .p2align 2
factor:
    .word 3
```

```
$ masc assemble a.s -t linux-riscv64 -o a.o -v
a.s: 2 sections, 8 symbols, 10 relocations, written to a.o as elf
```

The ten relocations are `R_RISCV_HI20`, `R_RISCV_LO12_I`,
`R_RISCV_PCREL_HI20` and `R_RISCV_PCREL_LO12_I` for `la`, and
`R_RISCV_CALL_PLT`, each with an `R_RISCV_RELAX` beside it. With
`--no-relax --pic` there are five, none of them `R_RISCV_RELAX`, and `la`
becomes `R_RISCV_GOT_HI20` with its `R_RISCV_PCREL_LO12_I`. The object also
holds a `.riscv.attributes` section.

```
$ masc disassemble a.o -t linux-riscv64
	.globl scale
	.type scale, @function
	.size scale, 36
	.globl helper
	.globl full
	.section .text, "ax", @progbits
	.balign 2
scale:
	li t0, 1000
	mul a0, a0, t0
	lui a1, %hi(factor)
	lw a1, %lo(factor)(a1)
	add a0, a0, a1
	lla a2, factor
	call helper
	ret
full:
	.option push
	.option norvc
	addi a0, a0, 1
	ret
	.option pop
	.section .data, "aw", @progbits
	.balign 4
factor:
	.byte 0x03, 0x00, 0x00, 0x00
note[disasm.unwritten] the text does not state trait riscv-arch rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0, which the selection --isa names gives the assembler again (a.o)
```

## Targets

`linux-riscv64` defaults to the `lp64d` convention and the selection
`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0`.
`linux-riscv32` names neither a convention nor an instruction set, so it
needs `--abi` (`ilp32`, `ilp32f` or `ilp32d`) and `--isa`.
