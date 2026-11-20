# Extensions and feature selection

Every set has a catalog of extensions and levels. A selection names
extensions and levels, is closed over their implications once, and decides
which instructions and registers a statement may use. A level is a published
group of extensions under one name: an architecture version, as `armv9.2-a`
or `rv64gc`, a profile, as `rva23u64`, or a psABI level, as `x86-64-v3`.
`masc isas` lists each set's levels.

Execution state is kept apart from extensions as axes: the x86 code width
and `doitm`, the RISC-V width and base, and AArch64's streaming mode, `dit`
and `nep`. A selection never names an axis value that code switches. Code
switches those itself, as `.code32` does.

## How each set spells a selection

| set     | spelling                                       | base, nothing added |
|---------|------------------------------------------------|---------------------|
| x86     | comma separated names, as `x86-64-v3,avx2`     | `i386`              |
| AArch64 | a level and `+` and `-` modifiers, as `armv8.2-a+fp+simd+nolse` | |
| RISC-V  | an isa string, as `rv32imac` or `rv64i2p1_m2p0_zbb1p0` | |
| wasm    | comma separated names, as `wasm2,tail-call`    | `mvp`               |
| SPIR-V  | a version and capabilities, as `spv1.6,Shader` | |

x86 extension names are lower case with underscores, as `avx512f`,
`amx_tile` and `apx_f`. A list of extensions alone is taken over the base,
as `--isa cmov,sse2` on `linux-x86`, and `i386` is the base alone:
`--isa i386,cmov,sse2` is refused. AArch64 levels select their version, which closes to
every feature the version makes mandatory, so `armv8-a` admits no
floating-point instruction and `armv8-a+fp+simd` does. A feature's revision
is not enforced, so `+sve` is admitted at `armv8-a`. A RISC-V isa string is
read and spelled in the canonical order of the isa naming grammar.

An unknown name is refused, and the refusal lists every name the set takes:

```
$ masc assemble avx.s -t linux-x86_64 --ext nonesuch -o avx.o
error[target] the architecture has no extension named `nonesuch`
  @ target, ext
  note it accepts 3dnow, ace, adx, aes, amx_avx512, ...
```

## Choosing from the command line

The target names a default selection. `--isa` replaces it, and each `--ext`
adds one extension to it:

```
$ masc assemble avx.s -t linux-x86_64 -o avx.o
error[asm.refused] the selection does not make this register available (needs avx)
  @ avx.s:3:12
3 |     vaddps ymm0, ymm0, ymm1
  |            ^^^^
$ masc assemble avx.s -t linux-x86_64 --ext avx -o avx.o
$ masc assemble avx.s -t linux-x86_64 --isa x86-64-v3 -o avx.o -vv
avx.s: 1 sections, 1 symbols, 0 relocations, written to avx.o as elf
note[target] linux-x86_64, abi sysv64, isa cx8,cmov,x87,fxsr,osfxsr,mmx,sce,sse,sse2,sse3,ssse3,sse41,sse42,cx16,lahf,popcnt,movbe,xsave,osxsave,bmi1,bmi2,lzcnt,avx,f16c,fma,avx2, format elf
```

The refusal names what is missing in parentheses, as `(needs avx)`,
`(needs zca)`, `(needs FEAT_DIT)` or `(needs sign-ext)`.

Some systems bound the machine not at all, and their targets name no
selection of their own. `linux-x86` and `linux-riscv32` need `--isa`.

`disassemble` and `walk` take the same flags, and decode under the selection
they give.

## Choosing in the source

| set     | directive                                              |
|---------|--------------------------------------------------------|
| x86     | `.isa selection` restates the whole selection          |
| AArch64 | `.arch level+mods` restates it by an architecture level, `.arch_extension crc` adds one, `.arch_extension nocrc` removes it with every extension that implies it |
| RISC-V  | `.option arch, spelling` restates or edits it, `.option rvc` and `.option norvc` add and remove the compressed extensions, `.option push` and `.option pop` save and restore it |
| SPIR-V  | `OpCapability` and `OpExtension` widen it for what follows |
| wasm    | none                                                   |

A selection a source states holds for the statements after it. The object
keeps its own selection, which the target gave, and a listing restates a
wider view around the statements written under it:

```
$ masc listing isa2.s -t linux-x86_64
	.intel_syntax noprefix
	.section .text, "ax", @progbits
	.isa cx8,cmov,x87,fxsr,osfxsr,mmx,sce,sse,sse2,sse3,ssse3,sse41,sse42,cx16,lahf,popcnt,movbe,xsave,osxsave,bmi1,bmi2,lzcnt,avx,f16c,fma,avx2
	vpaddd ymm0, ymm1, ymm2
	.isa cx8,cmov,x87,fxsr,osfxsr,mmx,sce,sse,sse2
```

## Extensions only a whole target may hold

An extension may be marked as one only a whole target's selection holds, as
RISC-V's `f` picks the calling convention's float registers. One function's
code may not select such an extension alone.
