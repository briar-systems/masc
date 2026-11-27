# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.2.2] - 2026-11-27

### Changed

- Requires mink `^0.2.0` and builds against mink 0.2.0, where 0.2.1 required `^0.1.0`.

### Fixed

- masc builds for a host whose `usize` is 32 bits: `masc.walk`'s `NO_OFFSET` is `usize`'s own all-ones and `masc.isa.wasm.object` holds a reserved pool index as the `usize` it is (#719).

## [0.2.1] - 2026-11-27

### Changed

- The command line writes mach's output: each command ends in a result block on stdout naming its verb and subject (`assembled`, `disassembled`, `listed`, `verified`, or `failed` once its work began and was refused), with its fields beneath and zero fields left out (#714).
- `-v` logs each stage with its time and detail, and `-vv` lists the resolved target's facts beneath the target line. Each diagnostic block is followed by a blank line (#714).
- `-q` leaves out results and notes and keeps errors and warnings, and `-q` with `-v` is refused as `cli.flag_conflict`. `walk` takes `-v` and `-q` (#714).
- `isas` prints its sets as a table (#714).

## [0.2.0] - 2026-11-26

### Added

- `masc.isa.wasm.object.import_named` states a wasm import from a module and field the object's names already hold, so a caller that lends its names to the object, as mirl does, no longer passes a view into names that interning the module can move (#715).

### Changed

- Requires mach `^7`, the 7.0.0 release and later, where 0.1.0 also admitted `7.0.0-rc.1`.

## [0.1.0] - 2026-11-23

### Added

- Initial release of `masc`, the multi-architecture assembler and CLI for Mach 7 targeting `x86_64`, `x86`, `aarch64`, `riscv64`, `riscv32`, WebAssembly (binary and WAT text format), and SPIR-V, with instruction-level emission, disassembly, and constant-time leakage analysis.
