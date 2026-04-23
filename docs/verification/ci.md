# CI

GitHub Actions workflow configuration lives in:

- [`.github/workflows/ci.yml`](https://github.com/QuAIR/0420-FPGA-Decoder/blob/main/.github/workflows/ci.yml)
- [`.github/workflows/docs.yml`](https://github.com/QuAIR/0420-FPGA-Decoder/blob/main/.github/workflows/docs.yml)

## CI Jobs

- `python-tests`: creates a conda environment named `fpga-decoder`, installs
  the `surface_code_decoder/` package in editable development mode, and runs the Python test
  suite from `surface_code_decoder/`.
- `coverage`: installs both Python packages, runs pytest coverage for
  `decoder_benchmark` and `qldpc`, writes package-local `coverage.xml` files,
  and uploads both reports to Codecov using OIDC authentication. Codecov upload
  failures are non-blocking so external provider outages do not hide the test
  and coverage generation result.
- `verilog-interface`: installs Icarus Verilog, installs the Python package,
  regenerates `verilog_test/vectors/current/`, runs the counter smoke test, and
  runs the decoder interface simulation.

If the Verilog job fails, the workflow uploads generated vectors and simulator
outputs as debug artifacts.

The coverage badge represents Python package coverage only. Verilog simulation
health is represented by the CI workflow badge unless HDL coverage is added in a
future workflow.

## Documentation Jobs

- `docs-links`: sets up Python and runs `python scripts/check_docs_links.py`
  from the repository root.

## Local Equivalents

Use [Surface Code Verilog Self-Check](surface-code-verilog.md) for the focused
local vector-generation and Verilog simulation commands.

Run Surface Code coverage from `surface_code_decoder/`:

```powershell
conda run -n fpga-decoder python -m pytest -q --cov=decoder_benchmark --cov-report=term-missing --cov-report=xml
```

On Windows, if pytest cannot access the default temporary directory, add a
fresh ignored basetemp path, for example
`--basetemp ..\pytest-temp-runs\issue8\decoder-coverage-local`.

Run QLDPC coverage from `qldpc_decoder/`:

```powershell
conda run -n fpga-decoder python -m pytest tests/ --cov=qldpc --cov-report=term-missing --cov-report=xml
```

Run the documentation link check from the repository root:

```powershell
python scripts\check_docs_links.py
```

Use [Environment](../development/environment.md) for setup notes.
