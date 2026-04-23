# GPU Decoder

`GPU-decoder/` is a parked future module. It is owned separately and is not
currently connected to the Surface Code Python-to-Verilog validation path or the
QLDPC decoder package.

The directory should not be treated as obsolete. It remains available for future
accelerator or backend integration work, but that integration should happen
through a dedicated design and implementation plan.

## Current Boundary

- Current Python-to-Verilog interface documentation lives under
  `docs/specs/interface/` and `docs/plans/interface/`.
- Current interface validation uses `surface_code_decoder/`, `verilog_test/`, scripts, and
  CI.
- Future GPU decoder integration should define its own ownership, API boundary,
  verification flow, and documentation links before being connected to the main
  validation path.
