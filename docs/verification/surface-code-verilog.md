# Surface Code Verilog Self-Check

This page documents the focused local validation flow for the Surface Code
Python-to-Verilog interface.

## Focused Python Tests

From `surface_code_decoder/`, run:

```powershell
conda run -n fpga-decoder python -m pytest tests/test_verilog_vectors.py tests/test_verilog_cli.py -q --basetemp ..\pytest-temp-runs\task1\run-local
```

If `surface_code_decoder/tests/test_verilog_vectors.py` is absent in the current worktree,
run the available CLI target instead:

```powershell
conda run -n fpga-decoder python -m pytest tests/test_verilog_cli.py -q --basetemp ..\pytest-temp-runs\task1\run-local
```

Use a fresh `--basetemp` subdirectory for repeated runs on Windows if older temp
directories are locked by the filesystem.

## Regenerate Verilog Vectors

From `surface_code_decoder/`, regenerate the vector bundle consumed by Verilog:

```powershell
conda run -n fpga-decoder python -m decoder_benchmark.verilog_cli --distance 3 --error-rate 0.001 --shots 4 --rounds 3 --seed 1234 --output-dir ..\verilog_test\vectors\current
```

The generated bundle should contain:

- `manifest.txt`
- `frame_map.txt`
- `syndrome_frames.mem`
- `expected_predictions.mem`
- `expected_observables.mem`
- `expected_summary.txt`

## Run Verilog Simulation

From the repository root:

```powershell
iverilog -o verilog_test\sim_decoder_interface.out verilog_test\decoder_interface_model.v verilog_test\tb_decoder_interface.v
vvp verilog_test\sim_decoder_interface.out
```

A healthy run prints PASS lines showing that all shots matched Python decoder
results and that aggregate logical-error and latency values matched.

## References

- [Python-Verilog Interface](../architecture/python-verilog-interface.md)
- [Environment](../development/environment.md)
- [Generated Artifacts](../development/generated-artifacts.md)
