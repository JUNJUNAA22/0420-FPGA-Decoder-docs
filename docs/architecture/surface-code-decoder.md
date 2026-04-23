# Surface Code Decoder

`surface_code_decoder/` is the Surface Code Python benchmark package. It builds rotated
surface-code memory circuits, samples detector data, decodes with the current
MWPM reference path, models latency, and writes reports and Verilog-friendly
vector bundles.

## Responsibilities

- Circuit generation through Stim-backed helpers.
- Syndrome sampling and frame grouping for streamed detector data.
- MWPM decoding through PyMatching.
- Logical-error counting and latency modeling.
- CSV, JSON, and Markdown reporting for benchmark runs.
- Verilog vector export for the interface-level simulation harness.

The Python package is the source of golden decoder data for the current
Python-to-Verilog interface flow. The Verilog side consumes exported files; it
does not run Python during simulation.

## References

- [Surface Code realtime MWPM FPGA design](../specs/surface-code/2026-04-20-surface-code-realtime-mwpm-fpga-design.md)
- [Python-Verilog Interface](python-verilog-interface.md)
- [Surface Code Verilog Self-Check](../verification/surface-code-verilog.md)
