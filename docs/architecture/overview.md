# Repository Overview

This repository is an FPGA quantum decoder exploration workspace. It is not a
single software package; it collects related decoder prototypes, simulation
harnesses, verification flows, and design records.

## Current Modules

- `surface_code_decoder/`: Surface Code Python benchmark package, including circuit
  generation, syndrome sampling, decoding, latency modeling, reporting, and
  Verilog vector export.
- `verilog_test/`: Verilog interface-level simulation harness for replaying
  exported vectors and checking contract-level decoder outputs.
- `qldpc_decoder/`: QLDPC adaptive sparse decoder module with Python APIs,
  sparse data structures, message passing, adaptive strategy work, and FPGA
  module skeletons.
- `GPU-decoder/`: parked future module owned separately and not currently wired
  into the main validation path.

The current root-level module locations describe the present repository state.
They are not a permanent source layout constraint. Future work may move
independently developed modules under a shared module or project directory in a
dedicated repository-layout migration.

## Module Details

- [Surface Code Decoder](surface-code-decoder.md)
- [Python-Verilog Interface](python-verilog-interface.md)
- [QLDPC Decoder](qldpc-decoder.md)
- [GPU Decoder](gpu-decoder.md)

For setup and verification commands, start with [Environment](../development/environment.md)
and [Surface Code Verilog Self-Check](../verification/surface-code-verilog.md).
