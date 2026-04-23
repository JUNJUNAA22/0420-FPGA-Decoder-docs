# Surface-Code Real-Time MWPM FPGA Simulation Design

## Goal

Build a simulation-first benchmark system for a real-time minimum-weight perfect matching decoder on the rotated planar surface code.

The first milestone is not RTL, HLS, or board-specific FPGA work. It is a Python report pipeline that establishes correctness and latency behavior under circuit-level noise while using module boundaries that can later map to FPGA pipeline stages.

The benchmark takes:

- Code distances `3`, `5`, and `7`
- A configurable physical error-rate sweep
- A configurable shot count
- A configurable number of syndrome rounds
- Explicit latency-model parameters

The benchmark produces:

- Logical error-rate measurements
- Host decode wall-time measurements
- Simulated decoder response latency measured in syndrome rounds
- Stage-level latency breakdowns suitable for later FPGA modeling

## Technology Choices

The first implementation will use:

- Python for orchestration, modeling, and reporting
- Stim for rotated planar surface-code memory circuits and circuit-level detector sampling
- PyMatching for the reference MWPM decoder
- sinter for baseline Stim/PyMatching Monte Carlo sampling where its API fits the benchmark flow

This stack keeps the first milestone focused on correctness and measurement. It also avoids committing to RTL before we understand the decoding workload and latency bottlenecks.

The implementation should reuse these packages instead of rebuilding their functionality. Custom code should focus on the real-time streaming representation, latency accounting, report packaging, and hardware-facing abstractions that are not provided by the baseline packages.

## Related Packages And Prior Art

The project should take advantage of existing QEC software instead of starting from scratch.

### Stim

Stim is the source of rotated planar surface-code memory circuits, detector samples, logical observables, and detector error models.

### PyMatching

PyMatching is the reference MWPM decoder. It can construct a matching graph directly from a Stim detector error model and decode detector samples.

### sinter

sinter combines Stim and PyMatching for fast parallel Monte Carlo sampling of QEC circuits. It records logical error statistics and writes CSV-style benchmark output.

Milestone one should use sinter when it cleanly supports the benchmark path. If the real-time streaming and latency model need lower-level control, the implementation may call Stim and PyMatching directly for that portion while still using sinter as a baseline cross-check.

### Micro Blossom

Micro Blossom is a hardware-accelerated exact MWPM decoder project with FPGA-oriented source code and graph-to-Verilog/VHDL generation.

Milestone one will not integrate Micro Blossom directly, but it should treat Micro Blossom as important prior art. Before inventing a custom FPGA graph representation, the implementation should inspect Micro Blossom's graph format and benchmark methodology.

### Fusion Blossom

Fusion Blossom is a fast MWPM decoder with streaming and parallel decoding relevance. It is optional for milestone one, but it is a useful comparison point for later decoder benchmarking.

### PanQEC, qecsim, and qsurface

These packages are useful references for QEC simulation, geometry, visualization, and alternate decoder workflows. They are not core dependencies for milestone one because Stim, PyMatching, and sinter are better aligned with circuit-level surface-code memory benchmarks.

## Architecture

The system has five small components.

### CircuitFactory

`CircuitFactory` creates Stim rotated planar surface-code memory circuits for each distance, physical error rate, and round-count configuration.

It also extracts the detector error model used to construct the PyMatching decoder.

### SyndromeSampler

`SyndromeSampler` runs Stim shots and converts detector samples into a streaming representation.

Stim may produce batch arrays internally, but this layer presents the data as time-ordered syndrome frames so the rest of the system can be shaped like a real-time decoder.

### MWPMDecoder

`MWPMDecoder` wraps PyMatching.

It consumes syndrome events or frames and produces predicted observable flips. The first implementation may decode per shot or batch using PyMatching's efficient APIs, but the public interface should look like a decoder receiving streamed syndrome data.

### LatencyModel

`LatencyModel` tracks simulated real-time latency.

The first model assigns explicit costs to ingestion, graph update, matching, and correction output. It reports response delay in syndrome-round units and provides a stage-level breakdown.

The cost model must be replaceable so later milestones can model a more realistic FPGA pipeline without rewriting circuit generation, sampling, decoding, or reporting.

### ReportGenerator

`ReportGenerator` aggregates benchmark results into machine-readable artifacts and a readable report.

The first milestone should produce CSV, JSON, and Markdown. Plots are optional for the first milestone and can be added once the data format is stable.

When sinter is used for baseline sampling, `ReportGenerator` should ingest or mirror sinter-compatible statistics instead of defining an incompatible result format.

## Data Flow

The benchmark data flow is:

```text
Stim circuit
  -> detector error model
  -> PyMatching decoder

Stim circuit
  -> detector samples and observables
  -> time-ordered syndrome frames
  -> MWPM decode
  -> logical prediction
  -> correctness statistics and latency model
  -> benchmark artifacts
```

For baseline Monte Carlo runs, sinter may manage the Stim/PyMatching sampling loop. For latency-model runs, the project may use direct Stim/PyMatching calls so syndrome frames and per-stage latency accounting remain visible.

The streaming representation is part of the design even though the first PyMatching-backed decoder may still use non-incremental decoding internally. This keeps the software boundary aligned with the future FPGA accelerator boundary.

## Metrics

The first milestone reports the following metrics.

### Primary Metrics

`logical_error_rate`

The fraction of shots where the decoder's predicted observable flip disagrees with Stim's sampled observable.

`latency_rounds`

The simulated decoder response delay measured in syndrome-round units.

`latency_stage_breakdown`

The contribution of each modeled stage, initially ingestion, graph update, matching, and correction output.

### Secondary Metrics

`decode_wall_time`

Host CPU wall time spent decoding. This is useful as a software baseline, but it is not treated as FPGA latency.

`throughput_context`

Optional frames-per-second or shots-per-second measurements. These provide context, but the first milestone optimizes around latency per syndrome round rather than throughput.

## Benchmark Scope

The first benchmark sweep covers:

- Distances: `3`, `5`, and `7`
- Noise model: circuit-level depolarizing-style noise from Stim-generated rotated surface-code memory circuits
- Physical error rates: a configurable list, with initial defaults such as `0.001`, `0.003`, and `0.01`
- Rounds: default to `rounds = distance`, with command-line configurability for longer real-time streams

The default development run should be small enough for fast iteration, such as distance `3`, one physical error rate, and `100` shots.

The benchmark should include a baseline path that can be compared against sinter's standard output for at least one tiny configuration.

## Out Of Scope For Milestone One

Milestone one does not include:

- Actual RTL
- HLS implementation
- FPGA board selection
- Place-and-route
- A custom MWPM implementation
- Distributed matching
- Direct Micro Blossom integration
- Claims that PyMatching's internal algorithm directly maps to FPGA hardware

The FPGA-related output in milestone one is a timing and streaming abstraction around a reference MWPM decoder.

## Validation Plan

Validation happens in layers.

### Decoder Correctness

For each circuit configuration:

1. Generate the Stim circuit.
2. Extract the detector error model.
3. Build a PyMatching decoder.
4. Sample detector data and logical observables.
5. Decode detector samples.
6. Compare predicted observable flips against sampled observables.

This validates the logical error-rate path.

### Streaming Representation

`SyndromeSampler` must preserve sample content when converting batch detector arrays into time-ordered frames.

This does not make PyMatching incremental. It verifies that the real-time-facing representation is faithful to the original detector samples.

### Latency Model

Given a synthetic frame stream and fixed per-stage costs, `LatencyModel` must produce deterministic latency in syndrome-round units.

This keeps FPGA assumptions explicit, inspectable, and testable.

### Report Generation

A tiny benchmark run should produce internally consistent CSV, JSON, and Markdown outputs.

The initial smoke test should use distance `3`, one physical error rate, and a small shot count.

At least one tiny configuration should be cross-checked against a sinter-style baseline to make sure logical error counting is consistent.

## Acceptance Criteria

Milestone one is complete when one command can produce a benchmark bundle for distances `3`, `5`, and `7`.

The bundle must include:

- `results.csv` with one row per distance and noise configuration
- `results.json` with structured metrics and run metadata
- `report.md` with a readable summary table and brief interpretation

The implementation must keep the latency model separate from circuit generation, sampling, decoding, and reporting so future milestones can replace the simple model with a more realistic FPGA pipeline model.

The implementation must document which parts are delegated to Stim, PyMatching, and sinter, and which parts are custom.

## Known Limitations

PyMatching remains the reference MWPM engine in milestone one.

The first milestone models real-time data shape and latency accounting, but it does not prove that the full PyMatching algorithm can be implemented directly on FPGA.

The project should treat milestone-one latency numbers as a design aid, not hardware timing closure evidence.

## Next Step After Spec Approval

After this spec is reviewed and approved, the next step is to write an implementation plan with small, testable tasks.
