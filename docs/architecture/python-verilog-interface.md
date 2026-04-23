# Python-Verilog Interface

The Python-Verilog interface connects `surface_code_decoder/` with `verilog_test/` through a
plain text vector bundle. Python generates golden decoder data, and Verilog
replays that data through an interface-level model.

## Current Flow

1. `surface_code_decoder/` builds and samples a Surface Code circuit.
2. Python decodes the samples and exports a vector bundle.
3. The bundle is written to `verilog_test/vectors/current/`.
4. `verilog_test/tb_decoder_interface.v` reads the bundle.
5. The testbench drives frames into `verilog_test/decoder_interface_model.v`.
6. The testbench compares predictions, observables, logical-error counts, and
   latency values against Python-generated golden data.

## Boundary

`verilog_test/decoder_interface_model.v` is an interface contract model. It is
not a hardware MWPM implementation and does not reproduce PyMatching in RTL.
The current goal is to validate the stream shape, file contract, prediction
handshake, and aggregate summary values.

## Vector Bundle

The current bundle contains:

- `manifest.txt`
- `frame_map.txt`
- `syndrome_frames.mem`
- `expected_predictions.mem`
- `expected_observables.mem`
- `expected_summary.txt`

Generated vector contents are runtime artifacts. Regenerate them from Python
instead of treating them as long-term documentation.

## References

- [Python decoder to Verilog interface design](../specs/interface/2026-04-20-python-decoder-verilog-interface-design.md)
- [Python decoder to Verilog interface implementation plan](../plans/interface/2026-04-20-python-decoder-verilog-interface-implementation.md)
- [Surface Code Verilog Self-Check](../verification/surface-code-verilog.md)
