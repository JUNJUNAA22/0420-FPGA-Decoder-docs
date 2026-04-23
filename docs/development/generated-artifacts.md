# Generated Artifacts

Generated artifacts are useful for debugging and demos, but they are not
long-term documentation sources.

Do not commit generated runtime outputs such as:

- `surface_code_decoder/benchmark-out/`
- `verilog_test/vectors/current/`
- `pytest-temp-runs/`
- `.pytest-basetemp*`
- coverage outputs such as `.coverage`, `coverage.xml`, and `htmlcov/`
- Python cache directories such as `__pycache__/`
- simulator outputs such as `.out`, `.vcd`, `sim.out`, and
  `sim_decoder_interface.out`

Regenerate vector bundles from Python for normal development instead of
hand-editing generated files. If generated output is needed for human review,
document the command that produced it and keep the output local unless there is
a specific review reason to commit it.

## References

- [Environment](environment.md)
- [Surface Code Verilog Self-Check](../verification/surface-code-verilog.md)
