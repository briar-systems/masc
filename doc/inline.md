# Inline assembly

An inline block is what a front end hands masc for an `asm` block of a
function: text in the set's source dialect and syntax, with named operands
whose locations the register allocator chooses later. The library reads it
in three steps (`masc.inline`):

1. `parse` reads the block under the context the enclosing function runs
   under, in the dialect of the object format it will be encoded into, and
   refuses it with diagnostics at their spans in the caller's file.
2. `effects` answers the footprint of the whole block, its registers, flags,
   accesses, stack and control, which the allocator reads as it reads one
   instruction's.
3. `encode` writes the block through the builder once every operand has its
   location.

The `inline` command runs all three on one file, with each operand placed in
a register the command line names.

## The block

A block is one statement after another:

- instructions, in the set's syntax
- numbered labels, as `1:`, which the block's own branches name as `1b` and
  `1f`. Each becomes a label of the builder that no other block or function
  names
- the set's integer data directives, as `.word 0xd503201f`, for an encoding
  no row names. A data directive may declare the registers its bytes write,
  as `.byte 0x0f, 0x31 :: writes(rax, rdx)`
- a state directive of the dialect, as x86's `.code32`, for the statements
  after it. The block ends at the state it starts at

An operand is written `{name}` where an instruction takes a whole operand.
Only a name the consumer declares is an operand: any other braced text is the
syntax's own, as AArch64's `{v0.16b}` list. A memory operand's name stands
for the whole address, as AArch64's `ldr x1, {m}` for `[sp, #16]` or
RISC-V's `ld a0, {m}` for `16(sp)`.

`parse` reads each name as a probe of its constraint and refuses it where
the instruction does not admit every value the constraint does, so a
location the allocator later chooses within the constraint always encodes.

## The inline command

```
masc inline [options] <input> -t <sys-arch> -o <path>
```

Each `--operand name=register` declares an operand the block names, placed
in that register, read and written, its constraint the register's class.

```
    add {acc}, {step}
    jnz 1f
    xor {acc}, {acc}
1:
    rdtsc
```

```
$ masc inline blk.s -t linux-x86_64 --operand acc=rax --operand step=rcx -o blk.o -v
blk.s: 1 section, 0 symbols, 0 relocations, written to blk.o as elf
$ masc disassemble blk.o -t linux-x86_64
	.section .text, "ax", @progbits
	add rax, rcx
	jne .Ltmp0
	xor rax, rax
.Ltmp0:
	rdtsc
```

The block is encoded at the start of the object's first section.

A constraint the instruction does not admit, an operand the block names but
nobody declared, and a malformed declaration are refused:

```
$ masc inline blk.s -t linux-x86_64 --operand acc=eax --operand step=rcx -o blk2.o
error[inline.refused] the instruction does not admit every value of this operand's constraint here
  @ blk.s:1:16
1 |     add {acc}, {step}
  |                ^^^^^^
$ masc inline blk.s -t linux-x86_64 --operand acc=rax -o blk3.o
error[inline.refused] the block declares no operand of this name
  @ blk.s:1:16
1 |     add {acc}, {step}
  |                ^^^^^^
$ masc inline blk.s -t linux-x86_64 --operand acc -o blk3.o
error[inline.operand] an operand is declared as name=register, not 'acc'
$ masc inline blk.s -t linux-x86_64 --operand acc=r99 -o blk3.o
error[cli.register] the instruction set has no register 'r99'
```

Raw bytes with the registers they write:

```
    .byte 0x0f, 0x31 :: writes(rax, rdx)
```

```
$ masc inline raw.s -t linux-x86_64 -o raw.o -v
raw.s: 1 section, 0 symbols, 0 relocations, written to raw.o as elf
```
