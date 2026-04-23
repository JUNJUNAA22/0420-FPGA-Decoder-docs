# Python Decoder To Verilog Interface Simulation Design

## Goal

Build a simulation-first bridge between the existing Python `decoder` benchmark and a Verilog interface-level testbench.

The bridge must let Python export decoder results into a stable folder layout that a mainstream Verilog simulator can read without custom plugins, JSON parsers, or non-portable host integration. The Verilog side must replay streamed syndrome frames, emit per-shot prediction outputs, accumulate cross-shot statistics, and compare them against Python-generated golden data.

## Scope

This design covers:

- exporting benchmark data from the existing Python decoder into Verilog-friendly files
- defining a stable interchange folder layout and file formats
- defining an interface-level Verilog decoder model boundary
- defining a Verilog testbench that reads the exported files and validates per-shot and aggregate outputs

This design does not cover:

- implementing a hardware MWPM decoder
- reproducing PyMatching inside RTL
- FPGA synthesis or board integration
- DPI, PLI, socket, or directory-walking integrations

## Why This Boundary Fits The Existing Decoder

The current `decoder` project is a Python benchmark pipeline, not an RTL implementation. Its important outputs are:

- streamed syndrome frames derived from detector samples
- per-shot predicted observable flips
- per-shot sampled observables
- logical error counts and logical error rate
- modeled latency totals and stage breakdowns

These outputs already exist at the Python boundary and are sufficient to define a hardware-facing contract. The Verilog simulation should therefore validate the stream shape, prediction handshake, and summary statistics instead of attempting to reproduce the software MWPM internals.

## Architecture

The system has three parts.

### 1. Python Exporter

The Python side runs the existing benchmark flow and writes a Verilog vector bundle into a fixed directory.

Responsibilities:

- build the Stim circuit and detector error model using the current project code
- sample detector data and observables
- convert detector samples into ordered syndrome frames using `SyndromeSampler`
- decode the detector samples using `MWPMDecoder`
- compute logical error statistics and latency using the existing benchmark helpers
- export the resulting data into plain-text files for Verilog consumption

The exporter owns file generation only. It does not invoke the simulator.

### 2. Vector Bundle Directory

The vector bundle is a fixed folder that decouples Python execution from Verilog simulation.

Recommended location:

`verilog_test/vectors/current/`

The exporter always refreshes that directory. The Verilog testbench reads the same location every run, so the user can regenerate vectors without editing Verilog source.

### 3. Verilog Interface Simulation

The Verilog side contains:

- an interface-level decoder model module
- a file-driven testbench

The decoder model is a placeholder for a future RTL decoder and only represents the streaming contract and scoreboard-visible state. The testbench reads vector files, drives the interface, and validates outputs against Python golden results.

## Data Flow

The end-to-end flow is:

```text
Python decoder benchmark
  -> detector samples
  -> syndrome frames
  -> predictions
  -> observables
  -> logical error statistics
  -> latency statistics
  -> vector bundle files
  -> Verilog testbench
  -> streamed frame inputs
  -> interface-level decoder outputs
  -> PASS / FAIL comparison against Python golden data
```

The Verilog simulation consumes exported data only. It does not call Python at runtime.

## Vector Bundle Layout

The bundle directory contains the following files:

```text
verilog_test/
  vectors/
    current/
      manifest.txt
      frame_map.txt
      syndrome_frames.mem
      expected_predictions.mem
      expected_observables.mem
      expected_summary.txt
```

### `manifest.txt`

`manifest.txt` stores global dataset metadata as whitespace-separated key-value pairs.

Required keys:

- `VERSION`
- `DISTANCE`
- `SHOTS`
- `ROUNDS`
- `DETECTOR_COUNT`
- `OBSERVABLE_COUNT`
- `FRAMES_PER_SHOT`
- `LATENCY_SCALE`

Example:

```text
VERSION 1
DISTANCE 3
SHOTS 4
ROUNDS 3
DETECTOR_COUNT 24
OBSERVABLE_COUNT 1
FRAMES_PER_SHOT 3
LATENCY_SCALE 1000
```

`LATENCY_SCALE` converts floating-point latency into fixed-point integers so Verilog can compare integer values instead of real numbers.

### `frame_map.txt`

`frame_map.txt` defines the meaning of each frame slot within a shot.

Each line contains:

```text
<frame_index> <round_index> <active_mask_hex>
```

Example:

```text
0 0 00FF
1 1 0F0F
2 2 F000
```

The active mask indicates which detector bits are valid in that frame. This mirrors the Python `SyndromeFrame` grouping behavior, where one frame only covers a subset of detectors.

### `syndrome_frames.mem`

`syndrome_frames.mem` stores frame data in shot-major order:

```text
shot0_frame0
shot0_frame1
shot0_frame2
shot1_frame0
shot1_frame1
shot1_frame2
...
```

Each line is a full-width hexadecimal detector vector. Bits outside the active mask must be zero.

### `expected_predictions.mem`

`expected_predictions.mem` stores the Python decoder prediction for each shot. Each line is one observable vector.

### `expected_observables.mem`

`expected_observables.mem` stores the sampled observable vector for each shot. Each line is one observable vector.

### `expected_summary.txt`

`expected_summary.txt` stores aggregate statistics as key-value pairs.

Required keys:

- `LOGICAL_ERRORS`
- `LOGICAL_ERROR_RATE_NUM`
- `LOGICAL_ERROR_RATE_DEN`
- `LATENCY_TOTAL_SCALED`
- `LATENCY_INGESTION_SCALED`
- `LATENCY_GRAPH_UPDATE_SCALED`
- `LATENCY_MATCHING_SCALED`
- `LATENCY_CORRECTION_OUTPUT_SCALED`

Example:

```text
LOGICAL_ERRORS 1
LOGICAL_ERROR_RATE_NUM 1
LOGICAL_ERROR_RATE_DEN 4
LATENCY_TOTAL_SCALED 3250
LATENCY_INGESTION_SCALED 1000
LATENCY_GRAPH_UPDATE_SCALED 1000
LATENCY_MATCHING_SCALED 1000
LATENCY_CORRECTION_OUTPUT_SCALED 250
```

The logical error rate is stored as numerator and denominator to avoid floating-point arithmetic in Verilog.

## Export Rules

The Python exporter must preserve the decoder semantics already defined in the repository.

### Frame Semantics

- frames for a shot are ordered by nondecreasing `round_index`
- together, frames cover every detector exactly once within a shot
- bits set outside the exported active mask are forbidden

### Prediction Semantics

- one exported prediction row corresponds to one shot
- prediction width matches the circuit observable count
- the exported prediction must equal the `MWPMDecoder.decode_batch` result for that shot

### Logical Error Semantics

Logical errors are counted exactly the same way as `count_logical_errors(...)` in Python:

- compare prediction and observable row by row
- if any observable bit differs for a shot, that shot counts as one logical error

### Latency Semantics

Latency remains a Python-modeled reference value, not a timing result discovered by RTL execution.

The exported latency fields must therefore reflect the Python `LatencyModel` output for the corresponding frame count:

- ingestion = `ingestion_cost * frame_count`
- graph update = `graph_update_cost * frame_count`
- matching = `matching_cost`
- correction output = `correction_output_cost`
- total = sum of stage values

All exported latency values are scaled integers.

## Verilog Module Boundary

The interface-level DUT is a contract model for future hardware, not a PyMatching reimplementation.

Recommended module name:

`decoder_interface_model`

Recommended parameters:

- `DETECTOR_W`
- `OBS_W`
- `ROUND_W`
- `COUNT_W`

Recommended inputs:

- `clk`
- `rst_n`
- `shot_start`
- `frame_valid`
- `shot_last_frame`
- `frame_round_idx`
- `frame_active_mask`
- `frame_syndrome_bits`
- `expected_prediction_valid`
- `expected_prediction_bits`
- `observable_valid`
- `observable_bits`
- `latency_total_scaled`
- `latency_ingestion_scaled`
- `latency_graph_update_scaled`
- `latency_matching_scaled`
- `latency_correction_scaled`

Recommended outputs:

- `prediction_valid`
- `prediction_bits`
- `shots_processed`
- `logical_error_count`
- `reported_latency_total_scaled`
- `reported_latency_ingestion_scaled`
- `reported_latency_graph_update_scaled`
- `reported_latency_matching_scaled`
- `reported_latency_correction_scaled`

## Verilog Testbench Behavior

Recommended testbench name:

`tb_decoder_interface`

The testbench owns all file-system interaction and golden-result checking.

### Initialization

At startup, the testbench:

- opens and parses `manifest.txt`
- opens and parses `frame_map.txt`
- loads `.mem` files containing frames, predictions, and observables
- opens and parses `expected_summary.txt`
- validates that all required files and keys exist

### Stimulus Sequence

The testbench drives one frame per cycle using a simple, readable timing model:

1. assert `shot_start` for the first frame of a shot
2. drive `frame_valid`, `frame_round_idx`, `frame_active_mask`, and `frame_syndrome_bits`
3. assert `shot_last_frame` on the final frame of the shot
4. on the next cycle, provide `expected_prediction_valid` and `observable_valid`
5. wait for `prediction_valid`
6. compare the emitted `prediction_bits` with the golden prediction for that shot
7. repeat for the next shot

This simple cadence is intentionally easy to inspect in waveforms and easy to replace later with a real decoder that has internal latency.

### Final Checks

After all shots complete, the testbench compares:

- processed shot count
- logical error count
- scaled latency totals
- scaled latency stage breakdowns

If every check passes, the testbench prints a clear PASS summary. Otherwise it prints a clear FAIL summary with shot indices and mismatched values.

## Error Handling

The testbench must stop early and loudly on malformed input data.

### File-Level Errors

If a required file cannot be opened or read, the testbench prints an error and terminates.

### Manifest Errors

The testbench rejects invalid metadata such as:

- `SHOTS < 1`
- `FRAMES_PER_SHOT < 1`
- `DETECTOR_COUNT < 1`
- `OBSERVABLE_COUNT < 1`
- `LATENCY_SCALE < 1`

### Frame Errors

The testbench rejects frame streams that violate the exported contract, including:

- frame index out of range
- decreasing round indices within a shot
- all-zero active mask
- syndrome bits set outside the active mask

### Output Mismatches

The testbench reports:

- per-shot prediction mismatches
- aggregate logical error mismatches
- aggregate latency mismatches

Each error message must include enough information to identify the failing shot or field.

## Compatibility Goals

The design deliberately favors common Verilog workflows over more advanced integration techniques.

The first implementation should therefore use:

- plain text metadata files
- `.mem` vector files
- `$fopen` and `$fscanf` for text parsing
- `$readmemh` for bulk vector loading
- a Verilog-2001-compatible coding style where practical

The design should avoid:

- directory traversal inside Verilog
- JSON parsing inside Verilog
- DPI or foreign-language runtime coupling
- simulator-specific APIs when a standard file-based flow is sufficient

## Testing Strategy

Validation should happen in two layers.

### Python Export Validation

Python tests should verify that the exporter:

- writes a complete vector bundle
- preserves detector frame ordering and active-mask meaning
- exports predictions and observables with correct widths
- exports summary values that match the current benchmark helpers

### Verilog Simulation Validation

The Verilog testbench should verify:

- every shot produces the expected prediction
- logical error accumulation matches the Python definition
- latency registers match the exported scaled summary values
- malformed vector bundles fail with explicit error messages

## Success Criteria

This design is successful when:

- a Python command can generate a fresh vector bundle under `verilog_test/vectors/current/`
- a Verilog simulation can read that bundle without source edits
- the simulation can validate per-shot predictions and aggregate results against Python golden data
- the file format remains stable when Python produces different benchmark runs
- the Verilog boundary remains reusable for a future real RTL decoder

## Known Limitations

This design does not prove that MWPM itself is implemented in hardware. It validates the interface contract around the current Python reference flow.

The first Verilog DUT is a contract model, not a true decoder core. Its value is in stabilizing the stream format, scoreboard rules, and summary reporting so later RTL can be dropped into the same harness.
