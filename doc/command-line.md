# Command line

```
masc <command> [options]
```

| command       | reads                         | writes                                  |
|---------------|-------------------------------|-----------------------------------------|
| `assemble`    | a source file                 | an object file (`-o`, required)         |
| `disassemble` | an object file                | source text, on stdout or to `-o`       |
| `listing`     | a source file                 | source text, on stdout or to `-o`       |
| `inline`      | one inline assembly block     | an object file (`-o`, required)         |
| `walk`        | an object file                | a `verified` result, or a refusal       |
| `isas`        | nothing                       | the registered sets as a table, on stdout |
| `help`        | nothing                       | the overview or one command's page      |

Each command except `isas` and `help` takes exactly one input file as its
operand. A missing input is refused before anything else runs, with
`error[io.missing] no file is at '<path>'`. The input is always a path:
`-` names a file called `-`, and so does `-o -`.

`masc` alone prints the overview and exits 1. `masc --version` prints the
version alone on one line. `masc help <command>`, `masc <command> --help` and
`masc <command> -h` print the page of that command and exit 0. `--help` shows
the page whatever operands the line holds, and `masc help` with a name that
is no command is refused with exit 1.

## Exit codes

| code | meaning                                                             |
|------|---------------------------------------------------------------------|
| 0    | the command did what was asked                                      |
| 1    | what was asked was refused: a bad line, a target, a source, an object |
| 2    | masc or a library it builds on broke an invariant                    |
| 3    | the system refused storage, a file or a stream                      |

A file that cannot be written ends with 3:

```
$ masc assemble a.s -t linux-x86_64 -o /nonexistent/dir/x.o
error[object.write] cannot write /nonexistent/dir/x.o (permission denied)

failed a.s
  target  linux-x86_64
  errors  1
  time    1.2ms
```

## Output

masc writes in the output language of mach, mirl and mink. A command that
writes something says so with a result on stdout, the same at every level:
the verb and what it acted on on one line, then one field a line beneath it,
indented two spaces, each key padded to the widest of the result and two
spaces before its value:

```
$ masc assemble a.s -t linux-x86_64 -o a.o
assembled a.s
  target    linux-x86_64
  sections  1
  symbols   1
  time      1.8ms
  output    a.o
```

The verbs are `assembled`, `disassembled`, `listed`, `verified` and
`failed`. A field with nothing to say is left out, so a count of zero is
never printed, and a path drops its leading `./`. A command whose work began
and did not do what was asked closes with `failed` in place of its own
result, with its target, the errors and warnings it reported and its time.
A refusal of the line itself, as a missing flag or a target the catalog does
not resolve, comes before any work and has no result. `disassemble` and
`listing` write their source to stdout as their product, and say so with a
result only when `-o` writes it to a file. `isas` is a listing, one table
under one header row, and has no result.

`-v` adds a log line on stderr for each stage as it closes: the seconds since
the command began, to a hundredth, the stage, its time and what it did. A
stage that reported errors or warnings counts them at the end of its line:

```
$ masc assemble a.s -t linux-x86_64 -o a.o -v
[  0.00] target       65us  linux-x86_64
[  0.00] parse       307us  4 lines
[  0.00] layout      1.4us  1 section
[  0.00] encode       15us  1 B
[  0.00] write       1.8ms  488 B
assembled a.s
  target    linux-x86_64
  sections  1
  symbols   1
  time      2.3ms
  output    a.o
```

| command       | stages                                                  |
|---------------|---------------------------------------------------------|
| `assemble`    | `target`, `parse`, `layout`, `encode`, `write`          |
| `inline`      | `target`, `parse`, `layout`, `encode`, `write`          |
| `listing`     | `target`, `parse`, `layout`, `encode`, `list`, `write` with `-o` |
| `disassemble` | `target`, `read`, `disasm`, `write` with `-o`           |
| `walk`        | `target`, `read`, `walk`                                |

`parse` runs from reading the source to the assembler's finish, less the time
the builder spends laying out and encoding, which close as `layout` and
`encode` after it. A format whose text states its module whole, as wasm, is
read in one step, so it logs `parse` and no `layout` or `encode`.

`-vv` adds the target's facts beneath its line: the instruction set selection
in force, extensions added with `--ext` included, spelled the way `--isa`
reads it back, then its system, environment, calling convention, code model,
linkage, object format and syntax, `-` where the target names none:

```
$ masc assemble b.s -t linux-x86 --isa i386 -o b.o -vv
[  0.00] target       49us  linux-x86
                            isa         i386
                            sys         linux
                            env         none
                            abi         sysv32
                            code_model  -
                            linkage     static
                            object      elf
                            syntax      intel
[  0.00] parse       619us  7 lines
[  0.00] layout       12us  1 section
[  0.00] encode       22us  13 B
[  0.00] write       1.2ms  416 B
assembled b.s
  target       linux-x86
  sections     1
  symbols      2
  relocations  1
  time         2.0ms
  output       b.o
```

`-q` leaves out the results and the notes, and keeps the errors and warnings.
`-q` and `-v` together are refused with `cli.flag_conflict`. masc writes
stderr before stdout, so the log lines and diagnostics read before the result
they lead to.

A duration is in microseconds below a millisecond, milliseconds below a
second and seconds below a minute, to a tenth while under ten of its unit,
then minutes and whole seconds. A size is exact in bytes below 1 KiB, then in
KiB, MiB or GiB to a tenth while under ten of the unit and whole above, as
`1.5 KiB` or `78 KiB`.

## Diagnostics

Every refusal is a block on stderr, the headline `error[<code>] <wording>`,
followed by its place when it is about text, and a blank line after the block
when anything follows it:

```
$ masc assemble avx.s -t linux-x86_64 -o avx.o
error[asm.refused] the selection does not admit the instruction (needs avx)
  @ avx.s:3:5
3 |     vaddps ymm0, ymm0, ymm1
  |     ^^^^^^

failed avx.s
  target  linux-x86_64
  errors  1
  time    499us
```

A refusal of the target names the part of it and lists what it accepts:

```
$ masc assemble a.s -t linux-x86_64 --env musl -o a.o
error[target] the system has no environment named `musl`
  @ target, env
  note it accepts gnu
```

The codes seen from the command line are `cli.line`, `cli.flag_missing`,
`cli.flag_conflict`, `cli.ext`, `cli.register`, `cli.memory`, `io.missing`, `io.read`,
`io.write`, `target`, `target.unserved`, `target.syntax`, `target.mode`,
`target.selection`, `object.read`, `object.write`, `object.format`,
`asm.refused`, `asm.mode`, `asm.cause`, `disasm.refused`, `listing.module`,
`listing.refused`, `inline.operand`, `inline.refused`, `inline.encode`,
`walk.symbol`, `walk.secret`, `walk.access`, `walk.private`, `walk.state`,
`walk.refused` and `walk.unread`. Notes are printed as `note[<code>]` and
warnings as `warning[<code>]`.

## The target flags

`assemble`, `disassemble`, `listing`, `inline` and `walk` all need a target.

| flag                      | meaning                                                  |
|---------------------------|----------------------------------------------------------|
| `-t`, `--target sys-arch` | the target, as `linux-x86_64`. required                   |
| `--env name`              | the environment, default the target's                    |
| `--abi name`              | the calling convention, default the target's             |
| `--isa selection`         | the instruction set selection, default the target's      |
| `--ext name`              | add one extension, repeatable, at most 64                |
| `--code-model name`       | the code model, default the target's                     |
| `--syntax name`           | the assembly syntax, default the set's one syntax        |

mink's target catalog resolves the target from these flags and checks every
narrowing it is given. masc then checks the instruction set selection against
the set that serves the architecture. The systems are `linux`, `darwin`,
`windows`, `uefi`, `freestanding`, `wasi`, `vulkan` and `opencl`, and each
system names the architectures it runs. The object format is the system's:
ELF on linux, COFF on windows, Mach-O on darwin, wasm on wasi and SPIR-V on
vulkan.

Some targets name no instruction set of their own and need `--isa`, and some
need `--abi`. The refusal says which and lists what it accepts:

```
$ masc assemble a.s -t linux-riscv32 -o a.o
error[target] the system's hosts for this architecture differ by calling convention, and the catalog cites no floor to derive one from. name the target's convention
  @ target, abi
  note it accepts ilp32, ilp32f, ilp32d
```

`linux-x86` and `linux-riscv32` both need `--isa` as well, as in
`masc assemble i386.s -t linux-x86 --isa i386 -o i386.o`. See
[extensions](extensions.md) for the selection spellings.

`-vv` lists the resolved target's facts beneath its log line, see
[output](#output). `walk` takes the target flags, `-v` and `-q`, but not `-o`.

## Mode flags

A set may declare assembler modes, each a `--[no-]<name>` flag. The flags of
every registered set are listed on `assemble` and `listing`, and a flag the
target's set does not declare is refused:

```
$ masc assemble b.s -t linux-x86_64 -o b.o --pic
error[target.mode] the set has no assembler mode the flag switches 'pic'
```

The modes declared today are RISC-V's:

| flag             | default | meaning                                                        |
|------------------|---------|----------------------------------------------------------------|
| `--[no-]pic`     | off     | read the source as position independent code, which decides what `la` loads |
| `--[no-]relax`   | on      | mark each relocation the linker may relax with `R_RISCV_RELAX` |

The source can switch both with `.option pic`, `.option nopic`, `.option
relax` and `.option norelax`.

## assemble

```
masc assemble [options] <input> -t <sys-arch> -o <path>
```

Reads the source file, assembles it under the target and writes the object
file in the target's format. Its result is `assembled`, with the target, the
object's sections, symbols and relocations, the warnings, the time and the
output.

The first refusal in the source ends the command with exit 1, a `failed`
result, and nothing written.

## disassemble

```
masc disassemble [options] <input> -t <sys-arch>
```

Reads an object file of any format mink reads, sniffed from its magic, and
prints it as source on stdout, or writes it to `-o`. `--no-aliases` prints
each instruction by its own form rather than its preferred alias. The code
is decoded under the target's selection, not under anything the source said,
so code assembled under a wider selection needs the same `--isa` or `--ext`
to disassemble as instructions. What the printed text cannot state of the
object is noted on stderr, see [disassembly](disassembly.md). With `-o` its
result is `disassembled`, with the target, the source's size, the warnings,
the time and the output:

```
$ masc disassemble a.o -t linux-x86_64 -o a.dis.s
disassembled a.o
  target  linux-x86_64
  size    61 B
  time    906us
  output  a.dis.s
```

## listing

```
masc listing [options] <input> -t <sys-arch>
```

Assembles the source as `assemble` does and prints the builder's record of
what it asked for, as source, on stdout or to `-o`. It writes no object. With
`-o` its result is `listed`, with the target, the listing's size, the
warnings, the time and the output. A format whose text states its module
whole, as wasm, is refused with `listing.module`. See
[disassembly](disassembly.md).

## inline

```
masc inline [options] <input> -t <sys-arch> -o <path>
```

Reads the file as one inline assembly block, places each operand it names in
the register `--operand name=register` gives, and writes the encoded block as
an object file. At most 16 operands, each name under 32 bytes. Its result is
`assembled`, as `assemble`'s. See [inline assembly](inline.md).

## walk

```
masc walk [options] <input> -t <sys-arch> --symbol <name>
```

Walks the function `--symbol` names in the object file from the seeds given,
and reports the first secret it leaks through timing. `--secret register`
seeds a register, `--access offset:n=secret|public` states what an access the
walk cannot place reads, and `--private offset:length` states a private
frame extent. At most 32 seeds in all. `--state axis=value`
sets a state value each instruction's timing is classified under, as
`--state dit=on`. A function that keeps its secrets constant time is
`verified`, with the target, the object, the warnings and the time:

```
$ masc walk ct.o -t linux-x86_64 --symbol sum --secret rdi --state doitm=on
verified sum
  target  linux-x86_64
  object  ct.o
  time    863us
```

See [constant-time walk](walk.md).

## isas

```
masc isas
```

Lists every registered instruction set as a table under one header row, a
row per set: the architectures it serves, its syntaxes with the established
one first and its levels, each comma-joined, `-` for none. It takes no
operand.

```
$ masc isas
arches                             syntaxes  levels
riscv64,riscv32,riscv64e,riscv32e  riscv     rv32i,rv64i,rv64gc,rva20u64,rva20s64,rva22u64,rva22s64,rva23u64,rva23s64,rvb23u64,rvb23s64
x86,x86_64                         intel     x86-64,x86-64-v2,x86-64-v3,x86-64-v4
wasm32,wasm64                      wat       wasm1,wasm2,wasm3
spirv                              spirv     -
aarch64,arm64e,arm64ec             aarch64   armv8-a,armv8.1-a,armv8.2-a,armv8.3-a,armv8.4-a,armv8.5-a,armv8.6-a,armv8.7-a,armv8.8-a,armv8.9-a,armv9-a,armv9.1-a,armv9.2-a,armv9.3-a,armv9.4-a,armv9.5-a,armv9.6-a,armv9.7-a,armv9.8-a
```
