# Environment

Use Python 3.11 or newer. The local development environment used for the
Surface Code flow is a conda environment named `fpga-decoder`.

## Python Setup

From `surface_code_decoder/`, install the Python package in editable mode:

```powershell
conda activate fpga-decoder
python -m pip install -e ".[dev]"
```

For one-shot commands, prefer explicit conda execution:

```powershell
conda run -n fpga-decoder python -m pytest -q
```

## Verilog Setup

The Verilog interface self-check expects `iverilog` and `vvp` on `PATH`.
Install Icarus Verilog or another compatible simulator before running the
Verilog commands.

## Windows Notes

PowerShell on this machine may print an execution-policy warning for the user
profile before command output. If the requested command still runs and returns
the expected exit code, do not treat that profile warning as a project failure.

Some local pytest temp directories may become locked by the Windows filesystem.
Use a fresh pytest `--basetemp` directory for repeated focused runs.

## Related Pages

- [Generated Artifacts](generated-artifacts.md)
- [Surface Code Verilog Self-Check](../verification/surface-code-verilog.md)
