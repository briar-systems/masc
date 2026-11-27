# WebAssembly text

The wasm set serves `wasm32` and `wasm64`. Its one syntax is `wat`, the
WebAssembly text format of release 3.0. The `wasi` system runs `wasm32`, and
its object format is a wasm module. No system of mink's catalog runs
`wasm64` yet:

```
$ masc assemble a.wat -t wasi-wasm64 -o a.wasm
error[target] the system does not run this architecture
  @ target, arch `wasm64`
  note it accepts wasm32
```

## The module text

The text is a module whole, `(module $id? field*)`, or its fields alone. Its
own reader takes it, not the statement reader of the machine sets, so the
directives of [the source page](source.md) do not apply. A line comment runs
from `;;`.

The reader takes the fields in passes, so a field may name what a later one
defines. An inline `(import "m" "n")` is the import it abbreviates, a table's
inline `(elem ...)` a table and an active segment, and a memory's inline
`(data ...)` a memory and an active segment.
Folded instructions are unfolded in the order they run.

Identifiers are `$` names. An index is its number or a `$` name its space
holds, and a label its depth or the name of a block around it. Integers are
decimal or after `0x`, with `_` between digits.

What the module's facts cannot state is refused: a type of the GC proposal
and a memory's page size.

## Example

```
;; a wasm module in the text format
(module
  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add)
  (func $twice (param i32) (result i32)
    (call $add (local.get 0) (local.get 0)))
  (memory 1)
  (data (i32.const 0) "hi"))
```

```
$ masc assemble a.wat -t wasi-wasm32 -o a.wasm
assembled a.wat
  target    wasi-wasm32
  sections  3
  time      1.6ms
  output    a.wasm
$ masc disassemble a.wasm -t wasi-wasm32
(module
  (type (func (param i32 i32) (result i32)))
  (type (func (param i32) (result i32)))
  (func (type 0)
    local.get 0
    local.get 1
    i32.add)
  (func (type 1)
    local.get 0
    local.get 0
    call 0)
  (memory 1)
  (data (memory 0) (offset
    i32.const 0) "hi"))
```

The printer writes every index by its number, each function's type by
`(type x)`, its instructions flat, one per line, and each segment in its full
form. The `$` names are not kept. Assembling the printed text again gives a
file of the same bytes.

A module of fields alone, with an import and a global:

```
(func $log (import "env" "log") (param i32))
(global $count (mut i32) (i32.const 0))
(func $tick
  global.get $count
  i32.const 1
  i32.add
  global.set $count
  (call $log (global.get $count)))
```

```
$ masc disassemble b.wasm -t wasi-wasm32 -q
(module
  (type (func (param i32)))
  (type (func))
  (import "env" "log" (func (type 0)))
  (func (type 1)
    global.get 0
    i32.const 1
    i32.add
    global.set 0
    global.get 0
    call 0)
  (global (mut i32)
    i32.const 0))
```

## Features

The features are those of the specification's release 3.0 and the threads
proposal, by their feature names, with the levels `wasm1`, `wasm2` and
`wasm3`. A selection is a comma separated list, as `wasm2,tail-call`.
`wasi-wasm32` defaults to `wasm1`:

```
$ masc assemble s.wat -t wasi-wasm32 -o s.wasm
error[asm.refused] the selection does not admit the instruction (needs sign-ext)
  @ s.wat:1:40
1 | (module (func (result i64) i64.const 1 i64.extend32_s))
  |                                        ^^^^^^^^^^^^^^

failed s.wat
  target  wasi-wasm32
  errors  1
  time    167us
```

`--isa wasm2` or `--ext sign-ext` admits it.

`listing` is refused for wasm, since the text states its module whole and no
builder lists it:

```
$ masc listing a.wat -t wasi-wasm32
error[listing.module] the format's text states its module whole, which no builder lists 'wasm'
```
