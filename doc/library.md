# Library

masc is a library first. Its declared public list is `src/lib/masc.mach`,
which forwards every module a dependent imports. A dependent imports a module
by its own path, as `use masc.build;`. Every module not on the list is
internal and may change without notice. The `masc-lib` artifact of
`mach.toml` builds the library, and the `masc` artifact the program.

The builder's frozen surface is `init`, `traits`, `make`, `switch`, `label`,
`define`, `expose`, `align`, `data`, `datum`, `finish`, `take`, `Size`, the
typed `Refusal`, `cfi` with its frame calls, and `ct_begin` and `ct_end`. It
grows by new calls and never by changing these.

## Emitting code

| module               | what it holds                                                        |
|----------------------|----------------------------------------------------------------------|
| `masc.build`         | the instruction-level builder: makes, finishes and reads a builder over one set and selection, into mink's neutral object |
| `masc.build.emit`    | the statements a section takes: instructions in their shortest admitted form, data, fills, alignment and values |
| `masc.build.frame`   | function frames and constant-time ranges stated at statements        |
| `masc.build.label`   | labels: naming, defining, exposing and equating them, and hand-stated relocations |
| `masc.build.refusal` | the typed refusal of a builder call and the wording of every defect  |
| `masc.build.section` | opening, grouping, ordering and switching sections                   |
| `masc.build.state`   | the builder's state record, its sections, fragments, labels and records |
| `masc.build.stats`   | what a builder measures of its work when asked                       |
| `masc.build.trait`   | the traits an object states, from the source and its instructions    |
| `masc.emit`          | code one instruction at a time by row and operands: `start`, an operand call per kind, `put`, and `sequence` for a pair under one relocation |
| `masc.emit.cfi`      | call frame facts as calls on the builder                             |
| `masc.emit.line`     | line facts: a file table and one location per run of instructions    |
| `masc.emit.listing`  | the builder's listing, source text that assembles alone to the same object |

## Inline assembly and walks

| module        | what it holds                                                                |
|---------------|------------------------------------------------------------------------------|
| `masc.inline` | inline blocks: `parse` at semantic analysis, `effects` for the allocator, `encode` after register allocation |
| `masc.walk`   | walks over encoded code by instruction effects: `check` and `check_timed`, the constant-time walk |

## Instruction sets and dialects

| module                | what it holds                                                       |
|-----------------------|---------------------------------------------------------------------|
| `masc.isa`            | the registry of sets, `find` by architecture, and `check` of a target mink resolved |
| `masc.format.dialect` | each object format's assembly dialect: private prefix, section grammar, symbol directives |
| `masc.codec`          | the encode and decode walks over a row's fields, and decode dispatch |

Per set, the list forwards the generated catalogs a compiler names rows,
registers and extensions by: for x86 `x86_gen_extension`,
`x86_gen_register`, `x86_gen_row`, `x86_gen_row_index` and `x86_set`, for
AArch64 `aarch64_field`, `aarch64_gen_extension`, `aarch64_gen_register`,
`aarch64_gen_row` and `aarch64_set`, for RISC-V the same four generated
modules, `riscv_mode` and `riscv_set`, for wasm its generated rows and
extensions with `wasm_body`, `wasm_fixup`, `wasm_kind`, `wasm_leb`,
`wasm_object`, `wasm_stream`, `wasm_text` and `wasm_valtype`, and for SPIR-V
`spirv_gen_extension`, `spirv_gen_operand`, `spirv_gen_row`,
`spirv_gen_row_index` and `spirv_kind`.

## The core

The core is the schema every set fills in.

| module                  | what it holds                                                      |
|-------------------------|--------------------------------------------------------------------|
| `masc.core.row`         | the row schema: operands, encoding fields, requirements, effect, timing class |
| `masc.core.operand`     | the operand model                                                  |
| `masc.core.register`    | the register model and register files                              |
| `masc.core.field`       | the field-kind contract every set fills in                         |
| `masc.core.inst`        | the encoded instruction value, a row and its operand values        |
| `masc.core.form`        | the encoding form, the choices no extension decides                |
| `masc.core.constraint`  | the register constraints of a row's operands                       |
| `masc.core.effect`      | the effect record every row references                             |
| `masc.core.footprint`   | an instruction's whole footprint: registers, flags, accesses, order, stack, transfer |
| `masc.core.query`       | queries over a row's effect given its operand values               |
| `masc.core.ct`          | the constant-time class, a set's timing, and the one refusal        |
| `masc.core.extension`   | extension catalogs and selections closed over them                 |
| `masc.core.requirement` | what a selection and state must hold for something to exist        |
| `masc.core.state`       | execution state axes and the context every reader takes            |
| `masc.core.switch`      | how code switches a state value itself                             |
| `masc.core.mode`        | assembler modes, how a set reads its source apart from what it selects |
| `masc.core.config`      | configuration state an instruction sets at run time                |
| `masc.core.quantity`    | quantities that scale with the implementation, as vector length    |
| `masc.core.fixup`       | how a set's fixup kinds bind to mink's relocation kinds            |
| `masc.core.relax`       | relaxation rows                                                    |
| `masc.core.invert`      | the inverse of each conditional branch row                         |
| `masc.core.dispatch`    | a set's codec record and decode dispatch                           |
| `masc.core.stream`      | the state a typed stream keeps across instructions                 |
| `masc.core.set`         | the registration record of a set                                   |
| `masc.core.index`       | the sorted indexes of tables looked up by name or number           |
| `masc.core.diagnostic`  | the one diagnostic of text masc reads                              |
| `masc.core.unmet`       | what a context leaves unmet of a requirement, and its wording      |

## Not on the list

The file assembler (`masc.asm`), the disassembler (`masc.disasm`), the
syntax readers and printers (`masc.syntax.*`) and the command line
(`masc.cli.*`, `masc.tool.*`) are not forwarded by the list, so they are
internal to masc. The `masc` program is built on them.
