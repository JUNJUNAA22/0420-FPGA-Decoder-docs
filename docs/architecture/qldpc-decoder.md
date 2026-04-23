# QLDPC Decoder

`qldpc_decoder/` is the QLDPC adaptive sparse decoder module. It is being
developed independently from the Surface Code interface flow and currently
contains Python data models, sparse formats, message passing, an initial decoder
API, adaptive strategy work, and FPGA module skeletons.

## Current Status

| Area | Status | Notes |
| --- | --- | --- |
| Package scaffold | Implemented | `pyproject.toml`, `src/qldpc/`, and tests are present. |
| Models | Implemented | Core dataclasses live in `src/qldpc/models.py`. |
| Sparse format | Implemented | Sparse matrix helpers live in `src/qldpc/sparse_format.py`. |
| Message passing | Implemented | The current decoder core lives in `src/qldpc/message_passing.py`. |
| Decoder API | Partial | `QLDPCDecoder` exists, but `decode_batch()` currently uses a simplified `H_x` path. |
| Adaptive strategy engine | Partial | Code analysis and strategy selection exist, with future runtime adaptation still expected. |
| Code generator | Planned | `src/qldpc/code_generator.py` exists, but integration tests still mark generation work as not complete. |
| FPGA host interface | Skeleton | `src/qldpc/fpga_interface.py` is present for future host integration. |
| Verilog FPGA modules | Skeleton | `src/fpga/` contains module skeletons that are not yet the current validation path. |
| End-to-end integration tests | Planned | `tests/test_integration.py` still contains skipped tests for the full pipeline. |
| Performance validation | Planned | Performance regression tests are not yet active. |

This status is intentionally conservative. Use current source files and tests as
the source of truth when judging readiness.

## References

- [QLDPC adaptive sparse decoder design](../specs/qldpc/2026-04-22-qldpc-adaptive-sparse-decoder-design.md)
- [QLDPC adaptive strategy engine design](../specs/qldpc/2026-04-22-qldpc-adaptive-strategy-engine-design.md)
- [QLDPC Decoder Phase 1 plan](../plans/qldpc/2026-04-22-qldpc-decoder-phase1-implementation.md)
- [QLDPC Adaptive Strategy Phase 2 plan](../plans/qldpc/2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md)
