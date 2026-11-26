# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `masc.isa.wasm.object.import_named` states a wasm import from a module and field the object's names already hold, so a caller that lends its names to the object, as mirl does, no longer passes a view into names that interning the module can move (#715).

## [0.1.0] - 2026-11-23

### Added

- Initial release of `masc`, the multi-architecture assembler and CLI for Mach 7 targeting `x86_64`, `x86`, `aarch64`, `riscv64`, `riscv32`, WebAssembly (binary and WAT text format), and SPIR-V, with instruction-level emission, disassembly, and constant-time leakage analysis.
