# Constant-time walk

The walk reads an encoded function through the set's decoder and follows
secrecy from the seeds it is given, through registers and memory, by each
instruction's effects. It refuses the first secret that reaches:

- a control transfer, as a conditional branch
- an address or an access length
- an input the instruction's timing depends on

## Effects and timing classes

Every row of every set references an effect record and a constant-time
class. The effect states what the instruction reads and writes beyond its
operands: implicit registers, flags, memory accesses, stack changes and where
control goes. The class states which inputs its timing depends on. A row is
dependent, independent, or independent under a condition, a requirement over
the set's atoms: an extension, as RISC-V's `zkt`, or a state value, as
AArch64's `dit=on` or x86's `doitm=on`.

Addresses, access lengths and branch conditions always matter, whatever the
class. The class covers timing alone.

A set states one of three things about its timing as a whole: it classifies
its rows (x86, AArch64, RISC-V), its timing is not guaranteed (wasm, whose
code a later compiler schedules), or it classifies none (SPIR-V). Code of a
set whose timing is not guaranteed is walked as any other, its rows' latency
is not asked, and a function it clears is answered with a warning. Code of a
set that classifies none is refused.

The refusal names the condition a row lacks in parentheses, as
`(needs zkt)`, `(needs dit=on)` or `(needs doitm=on)`.

## The walk command

```
masc walk [options] <input> -t <sys-arch> --symbol <name>
```

The function is the symbol `--symbol` names. It must be placed in a section
and state its size, as `.size name, . - name` gives it. The walk reads under
the target's selection and the state it enters at, and classifies each
instruction's timing under that state with the `--state` values set.

| flag                                 | seed                                     |
|--------------------------------------|------------------------------------------|
| `--secret register`                  | a register holding a secret on entry     |
| `--access offset:n=secret\|public`   | what the access of operand `n` of the instruction at `offset` reads |
| `--access offset:@n=secret\|public`  | the same, by the access's position `n` in the instruction's footprint |
| `--private offset:length`            | a frame extent off the stack pointer's value on entry that no unplaced access reaches |
| `--state axis=value`                 | a state value each instruction's timing is classified under, as `dit=on`; not a seed |

Offsets are decimal or hexadecimal after `0x`, from the function's start.
A `--private` offset may be negative, as `-16:16`, and its length is never
0. At most 32 seeds in all.

An access the walk cannot place, as a load through a pointer parameter, reads
a secret unless an `--access` seed says it reads public data.

| outcome            | output                                                  | exit |
|--------------------|---------------------------------------------------------|------|
| clear              | `<file>: <symbol>: constant time` on stdout              | 0    |
| unguaranteed       | `<file>: <symbol>: constant time but for the timing of its instructions` and a `warning[walk.unguaranteed]` | 0 |
| refused            | `error[walk.refused] <why>` and `@ <file>, <symbol> + <offset>` | 1 |
| unread             | `error[walk.unread] <why>` at the offset no row decodes  | 1    |

## RISC-V example

```
    .globl select
    .type select, @function
select:
    seqz t0, a0
    neg t0, t0
    xor t1, a1, a2
    and t1, t1, t0
    xor a0, a2, t1
    ret
    .size select, . - select

    .globl leaky
    .type leaky, @function
leaky:
    beqz a0, 1f
    mv a0, a1
1:  ret
    .size leaky, . - leaky
```

Without `zkt` no RISC-V row's timing is independent of its data:

```
$ masc walk ctrv.o -t linux-riscv64 --symbol select --secret a0
error[walk.refused] secret value in an input the instruction's timing depends on (needs zkt)
  @ ctrv.o, select + 0
```

With `zkt` in the selection, the branch-free select is clear and the branch
is refused:

```
$ masc assemble ctrv.s -t linux-riscv64 --ext zkt -o ctrvz.o
$ masc walk ctrvz.o -t linux-riscv64 --ext zkt --symbol select --secret a0
ctrvz.o: select: constant time
$ masc walk ctrvz.o -t linux-riscv64 --ext zkt --symbol leaky --secret a0
error[walk.refused] control transfer decided by a secret value
  @ ctrvz.o, leaky + 0
```

With no seed, `leaky` is clear:

```
$ masc walk ctrvz.o -t linux-riscv64 --ext zkt --symbol leaky
ctrvz.o: leaky: constant time
```

## Access seeds

```
    .globl check
    .type check, @function
check:
    ld a1, 0(a0)
    beqz a1, 1f
    li a0, 1
1:  ret
    .size check, . - check
```

The load at offset 0 goes through a pointer the walk cannot place, so what it
reads is secret and the branch on it is refused. Stating that operand 1 of
that load reads public data clears the function:

```
$ masc walk acc.o -t linux-riscv64 --ext zkt --symbol check
error[walk.refused] control transfer decided by a secret value
  @ acc.o, check + 2
$ masc walk acc.o -t linux-riscv64 --ext zkt --symbol check --access 0:1=public
acc.o: check: constant time
```

## Memory addresses

A secret that forms an address is refused on every set:

```
    .globl lookup
    .type lookup, @function
lookup:
    mov rax, qword ptr [rsi + rdi*8]
    ret
    .size lookup, . - lookup
```

```
$ masc walk ct.o -t linux-x86_64 --symbol lookup --secret rdi
error[walk.refused] memory access whose address or length is formed from a secret value
  @ ct.o, lookup + 0
```

## Timing state

Each instruction's timing is classified under the target's state on entry,
with each `--state axis=value` set on it. Decoding stays under the entry
state. x86's `doitm` is set only by the operating system, so on x86 every row
whose timing is independent only under `doitm=on` is refused unless the line
states `--state doitm=on`:

```
$ masc walk ct.o -t linux-x86_64 --symbol select --secret rdi
error[walk.refused] secret value in an input the instruction's timing depends on (needs doitm=on)
  @ ct.o, select + 0
```

The walk does not follow a switch the code makes. On AArch64 a function that
switches `dit` on with `msr dit, #1` is walked under the state the line
states, so without `--state dit=on` its first data-dependent row after the
switch is refused with `(needs dit=on)`:

```
select:
    msr dit, #1
    cmp x0, #0
    csel x0, x1, x2, eq
    ret
```

```
$ masc walk ct64.o -t linux-aarch64 --isa armv8.4-a --symbol select --secret x0
error[walk.refused] secret value in an input the instruction's timing depends on (needs dit=on)
  @ ct64.o, select + 4
```

With `--state dit=on` that row is classified under `dit=on`. A value that names no axis of the target, or
no value of one, is refused with `walk.state`. The library's
`walk.check_timed` takes the timing context from its caller, which is where a
compiler states `dit=on` for code it switches.
