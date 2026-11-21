# Assembly source

This page states what the machine sets (x86, AArch64, RISC-V) share. Each
set's own page states its comments, operands and directives. The WebAssembly
and SPIR-V texts are read by readers of their own, see [wasm](wasm.md) and
[spir-v](spirv.md).

## Statements

Source is a run of statements. A statement is a label, a directive, whose
name starts with a dot, or an instruction. A line holds any number of them.
Labels come before what follows them on the line, and the set's separator,
`;` on every machine set, ends a statement within a line.

```
"odd name": MOV RAX, 1 /* block comment */ ; RET
```

That one line is a label and two instructions. Mnemonics and register names
read without regard to case on every machine set, and disassembly prints them
in lower case.

## Comments

A line comment runs from the set's marker to the end of the line:

| set     | line comment |
|---------|--------------|
| x86     | `#`          |
| AArch64 | `//`         |
| RISC-V  | `#`          |
| wasm    | `;;`         |
| SPIR-V  | `;`          |

On the machine sets a comment may also run from `/*` to `*/`, across lines.
A comment that is not closed is refused.

## Names

A name is written bare when it is a letter, `_`, `.` or `$`, then letters,
digits, `_`, `.` and `$`. Any other name is written in double quotes, with `\`
before each `"` and `\` it holds. Those are the only two escapes a quoted name
takes.

A bare `.` is the place, the current offset. Digits then `b` or `f`, as `1b`
and `2f`, refer back or forward to the nearest numbered label `1:` or `2:`.
Written in quotes, `"."` and `"1b"` are symbols of those names.

Names that start with the dialect's private prefix are kept out of the
object's symbol table: `.L` on ELF, wasm, SPIR-V and COFF for x86-64 and
AArch64, `L` on Mach-O and on COFF for 32-bit x86. Labels the assembler and
the disassembler make with no name use the stem `.Ltmp` or `Ltmp` with a
number after it.

## Numbers, characters and strings

A number is decimal, hexadecimal after `0x`, binary after `0b`, octal after
`0o`, or octal after a leading `0`, so `010` is 8. Numbers are read up to 64
bits, as their bit pattern.

A character `'a'` is its byte. In a character `\n`, `\t`, `\r`, `\b`, `\f`
and up to three octal digits, as `'\101'`, are escapes, and a backslash
before any other byte is that byte. A string takes the same escapes and also
`\x` with hexadecimal digits, as `"\x41"`.

## Expressions

An operand or datum may be an expression of terms joined by binary operators,
with unary `-`, `+`, `~` and `!` and parentheses. A term is a number, a
character, a name, a numbered label reference or a relocation specifier.

| precedence | operators                                    |
|------------|----------------------------------------------|
| 4, tightest | `*` `/` `%` `<<` `>>`                       |
| 3          | `\|` `&` `^` `!` (or not: `a ! b` is `a \| ~b`) |
| 2          | `+` `-` `==` `!=` `<>` `<` `>` `<=` `>=`     |
| 1          | `&&` `\|\|`                                  |

A comparison is -1 when it holds and 0 when it does not. `&&` and `||` are 1
or 0. Every operator folds constants. Only `+` and `-` take a symbol: a
symbol plus or less a constant, or the difference of two symbols, which the
layout measures.

```
    .byte 3 > 2, 3 < 2, 1 <> 2, 2 == 2
    .byte 6 ! 1, ~0, -(4), 0b1010, 0o17, 017, 'a', '\n', '\101'
    .byte 7 % 4, 1 + 2 * 3, (1 + 2) * 3, 1 << 2 + 1, 5 && 0, 0 || 3
a:  .quad 0
b:  .quad b - a + 1
```

assembles to the bytes `ff 00 ff ff`, `fe ff fc 0a 0f 0f 61 0a 41`,
`03 07 09 05 00 01`, and two quads 0 and 9.

## Labels and symbols

A label `name:` defines a symbol at the place. A symbol is local until a
directive binds it otherwise:

| directive                      | meaning                                         |
|--------------------------------|-------------------------------------------------|
| `.globl name`, `.global name`  | bind globally. both take several names          |
| `.weak name`                   | bind weakly                                     |
| `.local name`                  | bind locally (ELF)                              |
| `.unique name`                 | bind as the one definition in the process (ELF `STB_GNU_UNIQUE`) |
| `.hidden`, `.internal`, `.protected` | set the visibility                        |
| `.type name, @kind`            | the kind, `@function` or `@object` (ELF). `%` may stand for `@` |
| `.size name, expr`             | the size (ELF)                                  |
| `.def name`, `.scl n`, `.type n`, `.endef` | a COFF symbol's class and type      |
| `.set name, expr`, `.equ name, expr` | equate a name to a value or symbol        |
| `.comm name, size, align`      | common storage                                  |
| `.lcomm name, size, align`     | local storage in `.bss`                          |
| `.weak_anti_dep name`          | a weak external searched as an anti-dependency (COFF), its fallback the name's `.set` |

COFF reads `.scl` 2 (external) and 3 (static) and `.type` 32 (function) and 0
(none). Mach-O holds no symbol kind or size, so its text states neither.

A symbol named in an operand and defined nowhere becomes an undefined global.

## Data, fill and alignment

| directive                          | meaning                                    |
|------------------------------------|--------------------------------------------|
| the set's integer data directives  | see each set's page for names and widths   |
| `.float x`, `.double x`            | IEEE 754 binary32 and binary64 values      |
| `.ascii "s"`                       | the string's bytes                         |
| `.asciz "s"`, `.string "s"`        | the string's bytes and a zero              |
| `.uleb128 n`, `.sleb128 n`         | LEB128 values                              |
| `.zero n`, `.skip n`               | n zero bytes                               |
| `.fill count, size, value`         | count copies of value, size bytes each     |
| `.align n`                         | align, in bytes on x86 and as a power of two on AArch64 and RISC-V |
| `.balign n`                        | align to n bytes                           |
| `.p2align n`                       | align to 2^n bytes                         |
| `.reloc offset, KIND, expr`        | a relocation stated by hand, by the format's name of its kind |

An alignment in a code section is filled with the set's no-op patterns.

## Sections

An assembly starts in `.text` (`__TEXT,__text` on Mach-O). The directives
that enter a section by its own name, which then takes the format's
convention for that name, are:

- ELF and COFF: `.text`, `.data`, `.bss`
- Mach-O: `.text`, `.const`, `.cstring`, `.literal4`, `.literal8`,
  `.literal16`, `.data`, `.const_data`, `.static_data`, `.mod_init_func`,
  `.mod_term_func`, `.bss`, `.tdata`

`.pushsection` and `.popsection` keep a stack of sections, and `.previous`
returns to the section before the current one.

`.section` takes the format's grammar after the section's name:

- ELF: `"flags", @type, entsize, linked, signature, comdat, unique, number`.
  A section with no stated flags takes ELF's convention for its name. A `G`
  flag with a signature and `comdat` puts the section in a comdat group, and
  an `o` flag with the name of a section orders it after that one. `unique,
  N` opens a section apart from the others of its name. A group that is no
  comdat group is refused.
- Mach-O: the name as `segment,section`, then `type, attribute+attribute`.
  The types are the S_ section types spelled in lower case, as `regular`,
  `zerofill` or `cstring_literals`, and the attributes read are
  `pure_instructions` and `debug`.
- COFF: `"flags", selection, symbol`. The flag letters are `x` code, `d`
  initialized data, `b` uninitialized data, `i` link information, `w`
  writable, `r` read only and `D` dropped by the link. The selections are
  `discard`, `one_only`, `same_size`, `same_contents`, `largest` and
  `associative`.

```
    .section .rodata, "a", @progbits
    .section .text.hot, "ax", @progbits, unique, 1
    .section .rdata, "dr"
    .cstring
```

`.subsections_via_symbols` states Mach-O's flag of the same name for the
whole file.

## Call frames, unwind codes and lines

The assembler reads `.cfi_*` call frame directives into the object's frame
records, which the format writes, as ELF's `.eh_frame`: `.cfi_startproc`,
`.cfi_endproc`, `.cfi_def_cfa`, `.cfi_def_cfa_offset`,
`.cfi_def_cfa_register`, `.cfi_adjust_cfa_offset`, `.cfi_offset`,
`.cfi_rel_offset`, `.cfi_val_offset`, `.cfi_register`, `.cfi_restore`,
`.cfi_undefined`, `.cfi_same_value`, `.cfi_remember_state`,
`.cfi_restore_state`, `.cfi_return_column`, `.cfi_signal_frame`,
`.cfi_escape`, `.cfi_personality`, `.cfi_lsda` and `.cfi_sections`.

```
framed:
    .cfi_startproc
    push rbp
    .cfi_def_cfa_offset 16
    .cfi_offset rbp, -16
    mov rbp, rsp
    .cfi_def_cfa_register rbp
    pop rbp
    .cfi_def_cfa rsp, 8
    ret
    .cfi_endproc
```

On linux-x86_64 that object holds an `.eh_frame` section with its relocation.

Windows unwind codes are the `.seh_*` directives: `.seh_proc`,
`.seh_endproc`, `.seh_endprologue`, `.seh_startepilogue`, `.seh_endepilogue`,
`.seh_handler`, `.seh_pushreg`, `.seh_setframe`, `.seh_stackalloc`,
`.seh_savereg`, `.seh_savexmm`, `.seh_pushframe`, and AArch64's
`.seh_save_reg`, `.seh_save_regp`, `.seh_save_fplr`, `.seh_set_fp`,
`.seh_add_fp`, `.seh_nop` and the rest of that family. `.seh_proc` names the
label placed where the procedure starts, so it follows that label:

```
f:
    .seh_proc f
    push rbp
    .seh_pushreg rbp
    sub rsp, 32
    .seh_stackalloc 32
    .seh_endprologue
    add rsp, 32
    pop rbp
    ret
    .seh_endproc
```

On windows-x86_64 that object holds `.xdata` and `.pdata` sections the format
writes from the frame.

`.file` and `.loc` state line records. A file of the table is numbered from
0, each directly after the last, as `.file 0 "/src" "p.c"`, and `.loc 0 3 0`
attaches line 3 of it to the next instruction. `.file "name"` with no number
names the source file the object is of.

## Relocations

A symbol in an operand becomes a relocation, its kind chosen by the operand
and the target: a call's target on x86-64 ELF is `R_X86_64_PLT32`, a
rip-relative address `R_X86_64_PC32`, a `.quad` naming a symbol
`R_X86_64_64`. A relocation specifier after or around the symbol chooses
another kind. The specifiers are each set's, listed on its page:

- x86: `sym@PLT`, `sym@GOTPCREL`, `sym@TPOFF` and the rest
- AArch64: `:lo12:sym`, `:got:sym` and the rest, and `sym@PAGE` and the rest
  on Mach-O
- RISC-V: `%hi(sym)`, `%lo(sym)`, `%pcrel_hi(sym)` and the rest

A specifier is read only under the formats whose dialect reads it.

## A whole example

```
# x86-64: sections, data, relocations, numbered labels, expressions
    .intel_syntax noprefix
    .text
    .globl count
    .type count, @function
count:
    xor eax, eax
1:
    cmp byte ptr [rdi], 0
    je 2f
    inc rdi; inc eax
    jmp 1b
2:
    call helper@PLT
    mov rcx, qword ptr [rip + table + 8]
    lea rdx, [rip + message]
    mov rsi, qword ptr [rip + extern_var@GOTPCREL]
    ret
    .size count, . - count

    .section .rodata, "a", @progbits
    .p2align 3
table:
    .quad count, helper
    .long 1 + 2 * 3, (1 << 4) | 0x0f, 'A', 010
message:
    .asciz "hi\n"
    .equ LIMIT, 64 - 8
    .byte LIMIT, LIMIT / 7, -1

    .data
    .weak helper
counter:
    .quad 0
    .bss
    .zero 16
    .comm shared, 32, 8
```

```
$ masc assemble b.s -t linux-x86_64 -o b.o -v
b.s: 4 sections, 8 symbols, 6 relocations, written to b.o as elf
```

The text section holds `R_X86_64_PLT32` against `helper`, `R_X86_64_PC32`
against `table` and `message`, and `R_X86_64_REX_GOTPCRELX` against
`extern_var`. The `.rodata` section holds two `R_X86_64_64`.
