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
| `walk`        | an object file                | a verdict on stdout, a refusal on stderr |
| `isas`        | nothing                       | the registered sets, on stdout          |
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
```

## Diagnostics

Every refusal is one headline on stderr, `error[<code>] <wording>`, followed
by its place when it is about text:

```
$ masc assemble avx.s -t linux-x86_64 -o avx.o
error[asm.refused] the selection does not make this register available (needs avx)
  @ avx.s:3:12
3 |     vaddps ymm0, ymm0, ymm1
  |            ^^^^
```

A refusal of the target names the part of it and lists what it accepts:

```
$ masc assemble a.s -t linux-x86_64 --env musl -o a.o
error[target] the system has no environment named `musl`
  @ target, env
  note it accepts gnu
```

The codes seen from the command line are `cli.line`, `cli.flag_missing`,
`cli.ext`, `cli.register`, `cli.memory`, `io.missing`, `io.read`,
`io.write`, `target`, `target.unserved`, `target.syntax`, `target.mode`,
`target.selection`, `object.read`, `object.write`, `object.format`,
`asm.refused`, `asm.mode`, `asm.cause`, `disasm.refused`, `listing.module`,
`listing.refused`, `inline.operand`, `inline.refused`, `inline.encode`,
`walk.symbol`, `walk.secret`, `walk.access`, `walk.private`,
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

`-v` reports what was written, and `-vv` also notes the resolved target on
stderr:

```
$ masc assemble b.s -t linux-x86_64 -o b.o -vv
b.s: 4 sections, 8 symbols, 6 relocations, written to b.o as elf
note[target] linux-x86_64, abi sysv64, isa cx8,cmov,x87,fxsr,osfxsr,mmx,sce,sse,sse2, format elf
```

The `isa` of that note is the selection in force, extensions added with
`--ext` included, spelled the way `--isa` reads it back. `-q` prints refusals
only, which drops the notes `disassemble` prints. `-q` and `-v` together are
refused.

`walk` takes the target flags but neither `-o`, `-v` nor `-q`.

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
file in the target's format. With `-v` it prints the counts of what it wrote:

```
$ masc assemble a.s -t linux-x86_64 -o a.o -v
a.s: 1 sections, 1 symbols, 0 relocations, written to a.o as elf
```

The first refusal in the source ends the command with exit 1 and nothing is
written.

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
object is noted on stderr, see [disassembly](disassembly.md). With `-o` and
`-v` it prints `<input>: <n> bytes of source, written to <path>`.

## listing

```
masc listing [options] <input> -t <sys-arch>
```

Assembles the source as `assemble` does and prints the builder's record of
what it asked for, as source, on stdout or to `-o`. It writes no object. A
format whose text states its module whole, as wasm, is refused with
`listing.module`. See [disassembly](disassembly.md).

## inline

```
masc inline [options] <input> -t <sys-arch> -o <path>
```

Reads the file as one inline assembly block, places each operand it names in
the register `--operand name=register` gives, and writes the encoded block as
an object file. At most 16 operands, each name under 32 bytes. See
[inline assembly](inline.md).

## walk

```
masc walk [options] <input> -t <sys-arch> --symbol <name>
```

Walks the function `--symbol` names in the object file from the seeds given,
and reports the first secret it leaks through timing. `--secret register`
seeds a register, `--access offset:n=secret|public` states what an access the
walk cannot place reads, and `--private offset:length` states a private
frame extent. At most 32 seeds in all. See [constant-time walk](walk.md).

## isas

```
masc isas
```

Lists every registered instruction set: the architectures it serves, its
syntaxes and its levels. It takes no operand.
