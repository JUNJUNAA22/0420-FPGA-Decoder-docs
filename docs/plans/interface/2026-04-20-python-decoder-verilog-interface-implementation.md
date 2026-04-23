# Python Decoder To Verilog Interface Implementation Plan

> Status: Historical implementation plan. The work described here may already
> be partially or fully implemented. Use the corresponding architecture page and
> current source tree as the source of truth.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Python exporter that writes Verilog-friendly decoder vectors and a self-checking Verilog harness that replays those vectors and validates per-shot plus aggregate results.

**Architecture:** Keep the Python benchmark and the Verilog simulation loosely coupled through plain-text files under `verilog_test/vectors/current/`. Add one Python export module plus one small CLI on the software side, then add one interface-level Verilog DUT and one file-driven testbench on the simulation side.

**Tech Stack:** Python 3.11, pytest, NumPy, Stim, PyMatching, Verilog-2001 style RTL/testbench, `iverilog`, `vvp`

---

## File Map

- Create `surface_code_decoder/src/decoder_benchmark/verilog_vectors.py`: export the current Python decoder results into `manifest.txt`, `frame_map.txt`, `.mem` vectors, and `expected_summary.txt`.
- Create `surface_code_decoder/src/decoder_benchmark/verilog_cli.py`: CLI entry point for generating `verilog_test/vectors/current/` bundles without hand-written scripts.
- Create `surface_code_decoder/tests/test_verilog_vectors.py`: unit tests for file layout, frame semantics, prediction rows, and scaled latency summary values.
- Create `surface_code_decoder/tests/test_verilog_cli.py`: tests for CLI defaults and end-to-end bundle generation.
- Modify `surface_code_decoder/pyproject.toml`: register the new `decoder-verilog-vectors` console script.
- Create `verilog_test/decoder_interface_model.v`: interface-level contract model that consumes frames, mirrors golden predictions, and accumulates counters.
- Create `verilog_test/tb_decoder_interface.v`: self-checking testbench that reads exported files, drives frames, compares predictions, and validates logical-error plus latency totals.
- Create `verilog_test/.gitignore`: ignore generated simulation artifacts and the mutable `vectors/current` contents.
- Create `verilog_test/vectors/.gitkeep`: keep the `vectors/` parent directory in the tree while allowing `current/` to remain generated output.

### Task 1: Add the Python Verilog vector exporter

**Files:**
- Create: `surface_code_decoder/tests/test_verilog_vectors.py`
- Create: `surface_code_decoder/src/decoder_benchmark/verilog_vectors.py`

Safety clarification:
`output_dir` reset means clearing the bundle output directory to a fresh top-level file set before rewrite. The exporter may delete top-level files inside `output_dir`, but it must not recursively delete nested directories.

- [ ] **Step 1: Write the failing test**

```python
from pathlib import Path

from decoder_benchmark.circuit_factory import CircuitFactory
from decoder_benchmark.decoder import MWPMDecoder, count_logical_errors
from decoder_benchmark.latency import LatencyModel
from decoder_benchmark.models import BenchmarkConfig, LatencyStageCosts
from decoder_benchmark.syndrome import SyndromeSampler
from decoder_benchmark.verilog_vectors import (
    DEFAULT_LATENCY_SCALE,
    export_verilog_vector_bundle,
)


def _read_key_value_file(path: Path) -> dict[str, str]:
    data: dict[str, str] = {}
    for raw_line in path.read_text().splitlines():
        line = raw_line.strip()
        if not line:
            continue
        key, value = line.split(maxsplit=1)
        data[key] = value
    return data


def _mask_from_indices(indices: tuple[int, ...]) -> int:
    mask = 0
    for index in indices:
        mask |= 1 << index
    return mask


def _bitvector_word(values) -> int:
    word = 0
    for bit_index, value in enumerate(values):
        if bool(value):
            word |= 1 << bit_index
    return word


def _frame_word(detector_indices: tuple[int, ...], values) -> int:
    word = 0
    for detector_index, value in zip(detector_indices, values, strict=True):
        if bool(value):
            word |= 1 << detector_index
    return word


def test_export_verilog_vector_bundle_writes_complete_bundle(tmp_path):
    config = BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=4, rounds=3)
    latency_costs = LatencyStageCosts(
        ingestion=0.25,
        graph_update=0.25,
        matching=1.0,
        correction_output=0.25,
    )

    paths = export_verilog_vector_bundle(
        config=config,
        output_dir=tmp_path,
        latency_costs=latency_costs,
        seed=1234,
    )

    assert paths.manifest_path.exists()
    assert paths.frame_map_path.exists()
    assert paths.syndrome_frames_path.exists()
    assert paths.expected_predictions_path.exists()
    assert paths.expected_observables_path.exists()
    assert paths.expected_summary_path.exists()

    factory = CircuitFactory()
    circuit = factory.build_memory_circuit(config)
    sampler = SyndromeSampler(seed=1234)
    manifest = _read_key_value_file(paths.manifest_path)

    assert manifest["VERSION"] == "1"
    assert manifest["DISTANCE"] == "3"
    assert manifest["SHOTS"] == "4"
    assert manifest["ROUNDS"] == "3"
    assert manifest["DETECTOR_COUNT"] == str(circuit.num_detectors)
    assert manifest["OBSERVABLE_COUNT"] == str(circuit.num_observables)
    assert manifest["FRAMES_PER_SHOT"] == str(len(sampler.detector_round_groups(circuit)))
    assert manifest["LATENCY_SCALE"] == str(DEFAULT_LATENCY_SCALE)


def test_exported_frames_and_summary_match_decoder_outputs(tmp_path):
    config = BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=4, rounds=3)
    latency_costs = LatencyStageCosts(
        ingestion=0.25,
        graph_update=0.25,
        matching=1.0,
        correction_output=0.25,
    )
    factory = CircuitFactory()
    circuit = factory.build_memory_circuit(config)
    sampler = SyndromeSampler(seed=1234)
    detector_samples, observable_samples = sampler.sample_batch(circuit, shots=config.shots)
    frame_groups = sampler.detector_round_groups(circuit)
    frames_by_shot = sampler.to_frames(circuit, detector_samples)
    decoder = MWPMDecoder.from_detector_error_model(factory.detector_error_model(circuit))
    predictions = decoder.decode_batch(detector_samples)
    logical_errors, _ = count_logical_errors(predictions, observable_samples)
    latency = LatencyModel(latency_costs).evaluate(frame_count=len(frame_groups))

    paths = export_verilog_vector_bundle(
        config=config,
        output_dir=tmp_path,
        latency_costs=latency_costs,
        seed=1234,
    )

    frame_map_rows = [line.split() for line in paths.frame_map_path.read_text().splitlines() if line.strip()]
    assert len(frame_map_rows) == len(frame_groups)
    assert [int(row[0]) for row in frame_map_rows] == list(range(len(frame_groups)))
    assert [int(row[1]) for row in frame_map_rows] == [group.round_index for group in frame_groups]
    assert [int(row[2], 16) for row in frame_map_rows] == [
        _mask_from_indices(group.detector_indices) for group in frame_groups
    ]

    syndrome_rows = [int(line, 16) for line in paths.syndrome_frames_path.read_text().splitlines() if line.strip()]
    expected_syndrome_rows = [
        _frame_word(frame.detector_indices, frame.values)
        for shot_frames in frames_by_shot
        for frame in shot_frames
    ]
    assert syndrome_rows == expected_syndrome_rows

    prediction_rows = [
        int(line, 16) for line in paths.expected_predictions_path.read_text().splitlines() if line.strip()
    ]
    observable_rows = [
        int(line, 16) for line in paths.expected_observables_path.read_text().splitlines() if line.strip()
    ]
    assert prediction_rows == [_bitvector_word(row) for row in predictions]
    assert observable_rows == [_bitvector_word(row) for row in observable_samples]

    summary = _read_key_value_file(paths.expected_summary_path)
    assert int(summary["LOGICAL_ERRORS"]) == logical_errors
    assert int(summary["LOGICAL_ERROR_RATE_NUM"]) == logical_errors
    assert int(summary["LOGICAL_ERROR_RATE_DEN"]) == config.shots
    assert int(summary["LATENCY_TOTAL_SCALED"]) == round(latency.latency_rounds * DEFAULT_LATENCY_SCALE)
    assert int(summary["LATENCY_INGESTION_SCALED"]) == round(
        latency.stage_breakdown["ingestion"] * DEFAULT_LATENCY_SCALE
    )
    assert int(summary["LATENCY_GRAPH_UPDATE_SCALED"]) == round(
        latency.stage_breakdown["graph_update"] * DEFAULT_LATENCY_SCALE
    )
    assert int(summary["LATENCY_MATCHING_SCALED"]) == round(
        latency.stage_breakdown["matching"] * DEFAULT_LATENCY_SCALE
    )
    assert int(summary["LATENCY_CORRECTION_OUTPUT_SCALED"]) == round(
        latency.stage_breakdown["correction_output"] * DEFAULT_LATENCY_SCALE
    )
```

- [ ] **Step 2: Run test to verify it fails**

Run from `surface_code_decoder/`: `python -m pytest tests/test_verilog_vectors.py -v`

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.verilog_vectors'`

- [ ] **Step 3: Write minimal implementation**

```python
from __future__ import annotations

import shutil
from dataclasses import dataclass
from pathlib import Path

import numpy as np

from decoder_benchmark.circuit_factory import CircuitFactory
from decoder_benchmark.decoder import MWPMDecoder, count_logical_errors
from decoder_benchmark.latency import LatencyModel
from decoder_benchmark.models import BenchmarkConfig, LatencyStageCosts
from decoder_benchmark.syndrome import DetectorRoundGroup, SyndromeFrame, SyndromeSampler

VECTOR_FORMAT_VERSION = 1
DEFAULT_LATENCY_SCALE = 1000
DEFAULT_VECTOR_OUTPUT_DIR = Path(__file__).resolve().parents[3] / "verilog_test" / "vectors" / "current"


@dataclass(frozen=True)
class VerilogVectorBundlePaths:
    manifest_path: Path
    frame_map_path: Path
    syndrome_frames_path: Path
    expected_predictions_path: Path
    expected_observables_path: Path
    expected_summary_path: Path


def export_verilog_vector_bundle(
    config: BenchmarkConfig,
    output_dir: str | Path = DEFAULT_VECTOR_OUTPUT_DIR,
    latency_costs: LatencyStageCosts | None = None,
    seed: int | None = None,
    latency_scale: int = DEFAULT_LATENCY_SCALE,
) -> VerilogVectorBundlePaths:
    if latency_scale < 1:
        raise ValueError("latency_scale must be at least 1")

    circuit_factory = CircuitFactory()
    circuit = circuit_factory.build_memory_circuit(config)
    sampler = SyndromeSampler(seed=seed)
    detector_samples, observable_samples = sampler.sample_batch(circuit, shots=config.shots)
    frame_groups = sampler.detector_round_groups(circuit)
    frames_by_shot = sampler.to_frames(circuit, detector_samples)

    decoder = MWPMDecoder.from_detector_error_model(circuit_factory.detector_error_model(circuit))
    predictions = decoder.decode_batch(detector_samples)
    logical_errors, _ = count_logical_errors(predictions, observable_samples)
    latency = LatencyModel(latency_costs).evaluate(frame_count=len(frame_groups))

    bundle_dir = Path(output_dir)
    _reset_output_dir(bundle_dir)
    paths = VerilogVectorBundlePaths(
        manifest_path=bundle_dir / "manifest.txt",
        frame_map_path=bundle_dir / "frame_map.txt",
        syndrome_frames_path=bundle_dir / "syndrome_frames.mem",
        expected_predictions_path=bundle_dir / "expected_predictions.mem",
        expected_observables_path=bundle_dir / "expected_observables.mem",
        expected_summary_path=bundle_dir / "expected_summary.txt",
    )

    manifest_lines = [
        f"VERSION {VECTOR_FORMAT_VERSION}",
        f"DISTANCE {config.distance}",
        f"SHOTS {config.shots}",
        f"ROUNDS {config.rounds}",
        f"DETECTOR_COUNT {circuit.num_detectors}",
        f"OBSERVABLE_COUNT {circuit.num_observables}",
        f"FRAMES_PER_SHOT {len(frame_groups)}",
        f"LATENCY_SCALE {latency_scale}",
    ]
    paths.manifest_path.write_text("\n".join(manifest_lines) + "\n")

    frame_map_lines = [
        f"{frame_index} {group.round_index} {_format_hex(_active_mask_word(group), circuit.num_detectors)}"
        for frame_index, group in enumerate(frame_groups)
    ]
    paths.frame_map_path.write_text("\n".join(frame_map_lines) + "\n")

    syndrome_rows = [
        _format_hex(_frame_word(frame), circuit.num_detectors)
        for shot_frames in frames_by_shot
        for frame in shot_frames
    ]
    paths.syndrome_frames_path.write_text("\n".join(syndrome_rows) + "\n")

    prediction_rows = [_format_hex(_bitvector_word(row), circuit.num_observables) for row in predictions]
    observable_rows = [_format_hex(_bitvector_word(row), circuit.num_observables) for row in observable_samples]
    paths.expected_predictions_path.write_text("\n".join(prediction_rows) + "\n")
    paths.expected_observables_path.write_text("\n".join(observable_rows) + "\n")

    summary_lines = [
        f"LOGICAL_ERRORS {logical_errors}",
        f"LOGICAL_ERROR_RATE_NUM {logical_errors}",
        f"LOGICAL_ERROR_RATE_DEN {config.shots}",
        f"LATENCY_TOTAL_SCALED {_scale_latency(latency.latency_rounds, latency_scale)}",
        f"LATENCY_INGESTION_SCALED {_scale_latency(latency.stage_breakdown['ingestion'], latency_scale)}",
        f"LATENCY_GRAPH_UPDATE_SCALED {_scale_latency(latency.stage_breakdown['graph_update'], latency_scale)}",
        f"LATENCY_MATCHING_SCALED {_scale_latency(latency.stage_breakdown['matching'], latency_scale)}",
        f"LATENCY_CORRECTION_OUTPUT_SCALED {_scale_latency(latency.stage_breakdown['correction_output'], latency_scale)}",
    ]
    paths.expected_summary_path.write_text("\n".join(summary_lines) + "\n")
    return paths


def _reset_output_dir(output_dir: Path) -> None:
    if output_dir.exists():
        if not output_dir.is_dir():
            raise ValueError("output_dir must point to a directory")
        for entry in output_dir.iterdir():
            if entry.is_dir():
                raise ValueError("output_dir must not contain nested directories")
            entry.unlink()
    else:
        output_dir.mkdir(parents=True, exist_ok=True)


def _scale_latency(value: float, latency_scale: int) -> int:
    return int(round(value * latency_scale))


def _format_hex(value: int, bit_width: int) -> str:
    hex_width = max(1, (bit_width + 3) // 4)
    return f"{value:0{hex_width}X}"


def _active_mask_word(group: DetectorRoundGroup) -> int:
    mask = 0
    for detector_index in group.detector_indices:
        mask |= 1 << detector_index
    return mask


def _frame_word(frame: SyndromeFrame) -> int:
    word = 0
    for detector_index, value in zip(frame.detector_indices, frame.values, strict=True):
        if bool(value):
            word |= 1 << detector_index
    return word


def _bitvector_word(values: np.ndarray) -> int:
    word = 0
    for bit_index, value in enumerate(np.asarray(values, dtype=np.bool_).tolist()):
        if value:
            word |= 1 << bit_index
    return word
```

- [ ] **Step 4: Run test to verify it passes**

Run from `surface_code_decoder/`: `python -m pytest tests/test_verilog_vectors.py -v`

Expected: PASS for both exporter tests

- [ ] **Step 5: Commit**

```bash
git add tests/test_verilog_vectors.py src/decoder_benchmark/verilog_vectors.py
git commit -m "feat: export decoder vectors for verilog"
```

### Task 2: Add a CLI for regenerating the current vector bundle

**Files:**
- Create: `surface_code_decoder/tests/test_verilog_cli.py`
- Create: `surface_code_decoder/src/decoder_benchmark/verilog_cli.py`
- Modify: `surface_code_decoder/pyproject.toml`

- [ ] **Step 1: Write the failing test**

```python
from pathlib import Path

from decoder_benchmark.verilog_cli import main, parse_args
from decoder_benchmark.verilog_vectors import DEFAULT_VECTOR_OUTPUT_DIR


def test_parse_args_defaults_to_current_vector_directory():
    args = parse_args(
        [
            "--distance",
            "3",
            "--error-rate",
            "0.001",
            "--shots",
            "4",
        ]
    )

    assert args.distance == 3
    assert args.error_rate == 0.001
    assert args.shots == 4
    assert args.rounds is None
    assert args.basis == "z"
    assert Path(args.output_dir) == DEFAULT_VECTOR_OUTPUT_DIR


def test_main_writes_verilog_vector_bundle(tmp_path):
    exit_code = main(
        [
            "--distance",
            "3",
            "--error-rate",
            "0.001",
            "--shots",
            "4",
            "--rounds",
            "3",
            "--seed",
            "1234",
            "--output-dir",
            str(tmp_path),
        ]
    )

    assert exit_code == 0
    assert (tmp_path / "manifest.txt").exists()
    assert (tmp_path / "frame_map.txt").exists()
    assert (tmp_path / "syndrome_frames.mem").exists()
    assert (tmp_path / "expected_predictions.mem").exists()
    assert (tmp_path / "expected_observables.mem").exists()
    assert (tmp_path / "expected_summary.txt").exists()
```

- [ ] **Step 2: Run test to verify it fails**

Run from `surface_code_decoder/`: `python -m pytest tests/test_verilog_cli.py -v`

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.verilog_cli'`

- [ ] **Step 3: Write minimal implementation**

```python
from __future__ import annotations

import argparse
from pathlib import Path
from typing import Sequence

from decoder_benchmark.models import BenchmarkConfig, LatencyStageCosts
from decoder_benchmark.verilog_vectors import (
    DEFAULT_LATENCY_SCALE,
    DEFAULT_VECTOR_OUTPUT_DIR,
    export_verilog_vector_bundle,
)


def parse_args(argv: Sequence[str] | None = None) -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        prog="decoder-verilog-vectors",
        description="Export Python decoder results into Verilog-friendly vector files.",
    )
    parser.add_argument("--distance", type=int, required=True)
    parser.add_argument("--error-rate", type=float, required=True)
    parser.add_argument("--shots", type=int, required=True)
    parser.add_argument("--rounds", type=int, default=None)
    parser.add_argument("--basis", choices=["x", "z"], default="z")
    parser.add_argument("--output-dir", default=str(DEFAULT_VECTOR_OUTPUT_DIR))
    parser.add_argument("--seed", type=int, default=None)
    parser.add_argument("--latency-scale", type=int, default=DEFAULT_LATENCY_SCALE)
    parser.add_argument("--ingestion-rounds", type=float, default=0.25)
    parser.add_argument("--graph-update-rounds", type=float, default=0.25)
    parser.add_argument("--matching-rounds", type=float, default=1.0)
    parser.add_argument("--correction-output-rounds", type=float, default=0.25)
    return parser.parse_args(argv)


def main(argv: Sequence[str] | None = None) -> int:
    args = parse_args(argv)
    config = BenchmarkConfig(
        distance=args.distance,
        physical_error_rate=args.error_rate,
        shots=args.shots,
        rounds=args.rounds,
        basis=args.basis,
    )
    latency_costs = LatencyStageCosts(
        ingestion=args.ingestion_rounds,
        graph_update=args.graph_update_rounds,
        matching=args.matching_rounds,
        correction_output=args.correction_output_rounds,
    )
    paths = export_verilog_vector_bundle(
        config=config,
        output_dir=Path(args.output_dir),
        latency_costs=latency_costs,
        seed=args.seed,
        latency_scale=args.latency_scale,
    )
    for path in (
        paths.manifest_path,
        paths.frame_map_path,
        paths.syndrome_frames_path,
        paths.expected_predictions_path,
        paths.expected_observables_path,
        paths.expected_summary_path,
    ):
        print(path)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

```toml
[project.scripts]
decoder-benchmark = "decoder_benchmark.cli:main"
decoder-verilog-vectors = "decoder_benchmark.verilog_cli:main"
```

- [ ] **Step 4: Run test to verify it passes**

Run from `surface_code_decoder/`: `python -m pytest tests/test_verilog_vectors.py tests/test_verilog_cli.py -v`

Expected: PASS for exporter and CLI tests

- [ ] **Step 5: Commit**

```bash
git add tests/test_verilog_cli.py src/decoder_benchmark/verilog_cli.py pyproject.toml
git commit -m "feat: add verilog vector export cli"
```

### Task 3: Add the Verilog interface contract model and self-checking testbench

**Files:**
- Create: `verilog_test/tb_decoder_interface.v`
- Create: `verilog_test/decoder_interface_model.v`
- Create: `verilog_test/.gitignore`
- Create: `verilog_test/vectors/.gitkeep`

- [ ] **Step 1: Write the failing test**

```verilog
`timescale 1ns/1ps

`define MANIFEST_FILE "verilog_test/vectors/current/manifest.txt"
`define FRAME_MAP_FILE "verilog_test/vectors/current/frame_map.txt"
`define SYNDROME_FILE "verilog_test/vectors/current/syndrome_frames.mem"
`define PREDICTION_FILE "verilog_test/vectors/current/expected_predictions.mem"
`define OBSERVABLE_FILE "verilog_test/vectors/current/expected_observables.mem"
`define SUMMARY_FILE "verilog_test/vectors/current/expected_summary.txt"

module tb_decoder_interface;
    localparam integer DETECTOR_W = 256;
    localparam integer OBS_W = 8;
    localparam integer ROUND_W = 8;
    localparam integer COUNT_W = 32;
    localparam integer MAX_SHOTS = 64;
    localparam integer MAX_FRAMES_PER_SHOT = 64;
    localparam integer MAX_TOTAL_FRAMES = MAX_SHOTS * MAX_FRAMES_PER_SHOT;

    reg clk;
    reg rst_n;
    reg shot_start;
    reg frame_valid;
    reg shot_last_frame;
    reg [ROUND_W-1:0] frame_round_idx;
    reg [DETECTOR_W-1:0] frame_active_mask;
    reg [DETECTOR_W-1:0] frame_syndrome_bits;
    reg expected_prediction_valid;
    reg [OBS_W-1:0] expected_prediction_bits;
    reg observable_valid;
    reg [OBS_W-1:0] observable_bits;
    reg [COUNT_W-1:0] latency_total_scaled;
    reg [COUNT_W-1:0] latency_ingestion_scaled;
    reg [COUNT_W-1:0] latency_graph_update_scaled;
    reg [COUNT_W-1:0] latency_matching_scaled;
    reg [COUNT_W-1:0] latency_correction_scaled;

    wire prediction_valid;
    wire [OBS_W-1:0] prediction_bits;
    wire [COUNT_W-1:0] shots_processed;
    wire [COUNT_W-1:0] logical_error_count;
    wire [COUNT_W-1:0] reported_latency_total_scaled;
    wire [COUNT_W-1:0] reported_latency_ingestion_scaled;
    wire [COUNT_W-1:0] reported_latency_graph_update_scaled;
    wire [COUNT_W-1:0] reported_latency_matching_scaled;
    wire [COUNT_W-1:0] reported_latency_correction_scaled;

    integer version;
    integer distance;
    integer shots;
    integer rounds;
    integer detector_count;
    integer observable_count;
    integer frames_per_shot;
    integer latency_scale_value;
    integer summary_logical_errors;
    integer summary_logical_error_rate_num;
    integer summary_logical_error_rate_den;
    integer summary_latency_total_scaled;
    integer summary_latency_ingestion_scaled;
    integer summary_latency_graph_update_scaled;
    integer summary_latency_matching_scaled;
    integer summary_latency_correction_scaled;
    integer error_count;
    integer computed_logical_errors;
    integer shot_index;
    integer frame_index;
    integer manifest_fd;
    integer frame_map_fd;
    integer summary_fd;
    integer parse_status;
    integer parsed_index;
    integer parsed_round;
    integer last_round_index;

    reg [DETECTOR_W-1:0] frame_active_masks [0:MAX_FRAMES_PER_SHOT-1];
    integer frame_rounds [0:MAX_FRAMES_PER_SHOT-1];
    reg [DETECTOR_W-1:0] syndrome_mem [0:MAX_TOTAL_FRAMES-1];
    reg [OBS_W-1:0] expected_prediction_mem [0:MAX_SHOTS-1];
    reg [OBS_W-1:0] expected_observable_mem [0:MAX_SHOTS-1];
    reg [DETECTOR_W-1:0] parsed_mask;
    reg [8*32-1:0] parsed_key;

    decoder_interface_model #(
        .DETECTOR_W(DETECTOR_W),
        .OBS_W(OBS_W),
        .ROUND_W(ROUND_W),
        .COUNT_W(COUNT_W)
    ) dut (
        .clk(clk),
        .rst_n(rst_n),
        .shot_start(shot_start),
        .frame_valid(frame_valid),
        .shot_last_frame(shot_last_frame),
        .frame_round_idx(frame_round_idx),
        .frame_active_mask(frame_active_mask),
        .frame_syndrome_bits(frame_syndrome_bits),
        .expected_prediction_valid(expected_prediction_valid),
        .expected_prediction_bits(expected_prediction_bits),
        .observable_valid(observable_valid),
        .observable_bits(observable_bits),
        .latency_total_scaled(latency_total_scaled),
        .latency_ingestion_scaled(latency_ingestion_scaled),
        .latency_graph_update_scaled(latency_graph_update_scaled),
        .latency_matching_scaled(latency_matching_scaled),
        .latency_correction_scaled(latency_correction_scaled),
        .prediction_valid(prediction_valid),
        .prediction_bits(prediction_bits),
        .shots_processed(shots_processed),
        .logical_error_count(logical_error_count),
        .reported_latency_total_scaled(reported_latency_total_scaled),
        .reported_latency_ingestion_scaled(reported_latency_ingestion_scaled),
        .reported_latency_graph_update_scaled(reported_latency_graph_update_scaled),
        .reported_latency_matching_scaled(reported_latency_matching_scaled),
        .reported_latency_correction_scaled(reported_latency_correction_scaled)
    );

    task automatic clear_inputs;
    begin
        shot_start = 1'b0;
        frame_valid = 1'b0;
        shot_last_frame = 1'b0;
        frame_round_idx = {ROUND_W{1'b0}};
        frame_active_mask = {DETECTOR_W{1'b0}};
        frame_syndrome_bits = {DETECTOR_W{1'b0}};
        expected_prediction_valid = 1'b0;
        expected_prediction_bits = {OBS_W{1'b0}};
        observable_valid = 1'b0;
        observable_bits = {OBS_W{1'b0}};
        latency_total_scaled = {COUNT_W{1'b0}};
        latency_ingestion_scaled = {COUNT_W{1'b0}};
        latency_graph_update_scaled = {COUNT_W{1'b0}};
        latency_matching_scaled = {COUNT_W{1'b0}};
        latency_correction_scaled = {COUNT_W{1'b0}};
    end
    endtask

    task automatic fail_now;
        input [8*64-1:0] message;
    begin
        $display("ERROR: %0s", message);
        $finish;
    end
    endtask

    task automatic read_named_int;
        input integer fd;
        input [8*32-1:0] expected_key;
        output integer value;
    begin
        parse_status = $fscanf(fd, "%s %d\n", parsed_key, value);
        if (parse_status != 2)
            fail_now("malformed key/value line");
        if (parsed_key != expected_key)
            fail_now("unexpected key order in metadata file");
    end
    endtask

    task automatic load_manifest;
    begin
        manifest_fd = $fopen(`MANIFEST_FILE, "r");
        if (manifest_fd == 0)
            fail_now("could not open manifest.txt");
        read_named_int(manifest_fd, "VERSION", version);
        read_named_int(manifest_fd, "DISTANCE", distance);
        read_named_int(manifest_fd, "SHOTS", shots);
        read_named_int(manifest_fd, "ROUNDS", rounds);
        read_named_int(manifest_fd, "DETECTOR_COUNT", detector_count);
        read_named_int(manifest_fd, "OBSERVABLE_COUNT", observable_count);
        read_named_int(manifest_fd, "FRAMES_PER_SHOT", frames_per_shot);
        read_named_int(manifest_fd, "LATENCY_SCALE", latency_scale_value);
        $fclose(manifest_fd);

        if (shots < 1)
            fail_now("SHOTS must be at least 1");
        if (frames_per_shot < 1)
            fail_now("FRAMES_PER_SHOT must be at least 1");
        if (detector_count < 1)
            fail_now("DETECTOR_COUNT must be at least 1");
        if (observable_count < 1)
            fail_now("OBSERVABLE_COUNT must be at least 1");
        if (latency_scale_value < 1)
            fail_now("LATENCY_SCALE must be at least 1");
        if (shots > MAX_SHOTS)
            fail_now("increase MAX_SHOTS in tb_decoder_interface.v");
        if (frames_per_shot > MAX_FRAMES_PER_SHOT)
            fail_now("increase MAX_FRAMES_PER_SHOT in tb_decoder_interface.v");
        if (shots * frames_per_shot > MAX_TOTAL_FRAMES)
            fail_now("increase MAX_TOTAL_FRAMES in tb_decoder_interface.v");
        if (detector_count > DETECTOR_W)
            fail_now("increase DETECTOR_W in tb_decoder_interface.v");
        if (observable_count > OBS_W)
            fail_now("increase OBS_W in tb_decoder_interface.v");
    end
    endtask

    task automatic load_frame_map;
    begin
        frame_map_fd = $fopen(`FRAME_MAP_FILE, "r");
        if (frame_map_fd == 0)
            fail_now("could not open frame_map.txt");
        last_round_index = -1;
        for (frame_index = 0; frame_index < frames_per_shot; frame_index = frame_index + 1) begin
            parse_status = $fscanf(frame_map_fd, "%d %d %h\n", parsed_index, parsed_round, parsed_mask);
            if (parse_status != 3)
                fail_now("malformed frame_map.txt line");
            if (parsed_index != frame_index)
                fail_now("frame index mismatch in frame_map.txt");
            if (parsed_mask == {DETECTOR_W{1'b0}})
                fail_now("active mask cannot be zero");
            if (parsed_round < last_round_index)
                fail_now("frame rounds must be nondecreasing");
            last_round_index = parsed_round;
            frame_rounds[frame_index] = parsed_round;
            frame_active_masks[frame_index] = parsed_mask;
        end
        $fclose(frame_map_fd);
    end
    endtask

    task automatic check_vector_file;
        input [8*64-1:0] path_name;
        input [8*48-1:0] human_name;
        integer fd;
    begin
        fd = $fopen(path_name, "r");
        if (fd == 0)
            fail_now(human_name);
        $fclose(fd);
    end
    endtask

    task automatic load_memories;
    begin
        check_vector_file(`SYNDROME_FILE, "could not open syndrome_frames.mem");
        check_vector_file(`PREDICTION_FILE, "could not open expected_predictions.mem");
        check_vector_file(`OBSERVABLE_FILE, "could not open expected_observables.mem");
        $readmemh(`SYNDROME_FILE, syndrome_mem);
        $readmemh(`PREDICTION_FILE, expected_prediction_mem);
        $readmemh(`OBSERVABLE_FILE, expected_observable_mem);
    end
    endtask

    task automatic load_summary;
    begin
        summary_fd = $fopen(`SUMMARY_FILE, "r");
        if (summary_fd == 0)
            fail_now("could not open expected_summary.txt");
        read_named_int(summary_fd, "LOGICAL_ERRORS", summary_logical_errors);
        read_named_int(summary_fd, "LOGICAL_ERROR_RATE_NUM", summary_logical_error_rate_num);
        read_named_int(summary_fd, "LOGICAL_ERROR_RATE_DEN", summary_logical_error_rate_den);
        read_named_int(summary_fd, "LATENCY_TOTAL_SCALED", summary_latency_total_scaled);
        read_named_int(summary_fd, "LATENCY_INGESTION_SCALED", summary_latency_ingestion_scaled);
        read_named_int(summary_fd, "LATENCY_GRAPH_UPDATE_SCALED", summary_latency_graph_update_scaled);
        read_named_int(summary_fd, "LATENCY_MATCHING_SCALED", summary_latency_matching_scaled);
        read_named_int(summary_fd, "LATENCY_CORRECTION_OUTPUT_SCALED", summary_latency_correction_scaled);
        $fclose(summary_fd);
    end
    endtask

    task automatic drive_frame;
        input integer current_shot;
        input integer current_frame;
        integer flat_index;
        reg [DETECTOR_W-1:0] invalid_bits;
    begin
        flat_index = (current_shot * frames_per_shot) + current_frame;
        invalid_bits = syndrome_mem[flat_index] & ~frame_active_masks[current_frame];
        if (invalid_bits != {DETECTOR_W{1'b0}}) begin
            error_count = error_count + 1;
            $display("ERROR: shot %0d frame %0d sets bits outside the active mask", current_shot, current_frame);
        end

        @(negedge clk);
        shot_start = (current_frame == 0);
        frame_valid = 1'b1;
        shot_last_frame = (current_frame == frames_per_shot - 1);
        frame_round_idx = frame_rounds[current_frame][ROUND_W-1:0];
        frame_active_mask = frame_active_masks[current_frame];
        frame_syndrome_bits = syndrome_mem[flat_index];
        @(posedge clk);
        @(negedge clk);
        clear_inputs();
    end
    endtask

    task automatic finish_shot;
        input integer current_shot;
    begin
        @(negedge clk);
        expected_prediction_valid = 1'b1;
        expected_prediction_bits = expected_prediction_mem[current_shot];
        observable_valid = 1'b1;
        observable_bits = expected_observable_mem[current_shot];
        latency_total_scaled = summary_latency_total_scaled;
        latency_ingestion_scaled = summary_latency_ingestion_scaled;
        latency_graph_update_scaled = summary_latency_graph_update_scaled;
        latency_matching_scaled = summary_latency_matching_scaled;
        latency_correction_scaled = summary_latency_correction_scaled;

        @(posedge clk);
        if (prediction_valid !== 1'b1) begin
            error_count = error_count + 1;
            $display("ERROR: shot %0d did not raise prediction_valid", current_shot);
        end
        if (prediction_bits !== expected_prediction_mem[current_shot]) begin
            error_count = error_count + 1;
            $display(
                "ERROR: shot %0d prediction mismatch expected=%0h actual=%0h",
                current_shot,
                expected_prediction_mem[current_shot],
                prediction_bits
            );
        end
        if (expected_prediction_mem[current_shot] != expected_observable_mem[current_shot])
            computed_logical_errors = computed_logical_errors + 1;

        @(negedge clk);
        clear_inputs();
    end
    endtask

    initial begin
        clk = 1'b0;
        forever #5 clk = ~clk;
    end

    initial begin
        error_count = 0;
        computed_logical_errors = 0;
        rst_n = 1'b0;
        clear_inputs();

        #20;
        rst_n = 1'b1;

        load_manifest();
        load_frame_map();
        load_memories();
        load_summary();

        for (shot_index = 0; shot_index < shots; shot_index = shot_index + 1) begin
            for (frame_index = 0; frame_index < frames_per_shot; frame_index = frame_index + 1)
                drive_frame(shot_index, frame_index);
            finish_shot(shot_index);
        end

        if (summary_logical_error_rate_den != shots) begin
            error_count = error_count + 1;
            $display("ERROR: summary denominator expected %0d shots but read %0d", shots, summary_logical_error_rate_den);
        end
        if (computed_logical_errors != summary_logical_errors) begin
            error_count = error_count + 1;
            $display(
                "ERROR: computed logical errors expected %0d but summary reports %0d",
                computed_logical_errors,
                summary_logical_errors
            );
        end
        if (shots_processed !== shots) begin
            error_count = error_count + 1;
            $display("ERROR: shots_processed expected %0d actual %0d", shots, shots_processed);
        end
        if (logical_error_count !== summary_logical_errors) begin
            error_count = error_count + 1;
            $display(
                "ERROR: logical_error_count expected %0d actual %0d",
                summary_logical_errors,
                logical_error_count
            );
        end
        if (reported_latency_total_scaled !== summary_latency_total_scaled) begin
            error_count = error_count + 1;
            $display(
                "ERROR: latency total expected %0d actual %0d",
                summary_latency_total_scaled,
                reported_latency_total_scaled
            );
        end
        if (reported_latency_ingestion_scaled !== summary_latency_ingestion_scaled) begin
            error_count = error_count + 1;
            $display(
                "ERROR: latency ingestion expected %0d actual %0d",
                summary_latency_ingestion_scaled,
                reported_latency_ingestion_scaled
            );
        end
        if (reported_latency_graph_update_scaled !== summary_latency_graph_update_scaled) begin
            error_count = error_count + 1;
            $display(
                "ERROR: latency graph_update expected %0d actual %0d",
                summary_latency_graph_update_scaled,
                reported_latency_graph_update_scaled
            );
        end
        if (reported_latency_matching_scaled !== summary_latency_matching_scaled) begin
            error_count = error_count + 1;
            $display(
                "ERROR: latency matching expected %0d actual %0d",
                summary_latency_matching_scaled,
                reported_latency_matching_scaled
            );
        end
        if (reported_latency_correction_scaled !== summary_latency_correction_scaled) begin
            error_count = error_count + 1;
            $display(
                "ERROR: latency correction expected %0d actual %0d",
                summary_latency_correction_scaled,
                reported_latency_correction_scaled
            );
        end

        if (error_count == 0) begin
            $display("PASS: all %0d shots matched Python decoder results", shots);
            $display(
                "PASS: logical_error_count=%0d latency_total_scaled=%0d",
                logical_error_count,
                reported_latency_total_scaled
            );
        end else begin
            $display("FAIL: %0d mismatches found", error_count);
        end
        $finish;
    end
endmodule
```

- [ ] **Step 2: Run test to verify it fails**

Run from the repo root:

`Push-Location decoder`

`$env:PYTHONPATH='src'`

`python -m decoder_benchmark.verilog_cli --distance 3 --error-rate 0.001 --shots 4 --rounds 3 --seed 1234 --output-dir ..\verilog_test\vectors\current`

`Pop-Location`

`iverilog -o verilog_test\sim_decoder_interface.out verilog_test\tb_decoder_interface.v`

Expected: FAIL at compile time because module `decoder_interface_model` is undefined

- [ ] **Step 3: Write minimal implementation**

```verilog
`timescale 1ns/1ps

module decoder_interface_model #(
    parameter DETECTOR_W = 256,
    parameter OBS_W = 8,
    parameter ROUND_W = 8,
    parameter COUNT_W = 32
)(
    input wire clk,
    input wire rst_n,
    input wire shot_start,
    input wire frame_valid,
    input wire shot_last_frame,
    input wire [ROUND_W-1:0] frame_round_idx,
    input wire [DETECTOR_W-1:0] frame_active_mask,
    input wire [DETECTOR_W-1:0] frame_syndrome_bits,
    input wire expected_prediction_valid,
    input wire [OBS_W-1:0] expected_prediction_bits,
    input wire observable_valid,
    input wire [OBS_W-1:0] observable_bits,
    input wire [COUNT_W-1:0] latency_total_scaled,
    input wire [COUNT_W-1:0] latency_ingestion_scaled,
    input wire [COUNT_W-1:0] latency_graph_update_scaled,
    input wire [COUNT_W-1:0] latency_matching_scaled,
    input wire [COUNT_W-1:0] latency_correction_scaled,
    output reg prediction_valid,
    output reg [OBS_W-1:0] prediction_bits,
    output reg [COUNT_W-1:0] shots_processed,
    output reg [COUNT_W-1:0] logical_error_count,
    output reg [COUNT_W-1:0] reported_latency_total_scaled,
    output reg [COUNT_W-1:0] reported_latency_ingestion_scaled,
    output reg [COUNT_W-1:0] reported_latency_graph_update_scaled,
    output reg [COUNT_W-1:0] reported_latency_matching_scaled,
    output reg [COUNT_W-1:0] reported_latency_correction_scaled
);
    reg shot_open;

    wire unused_signals;
    assign unused_signals = shot_last_frame ^ frame_valid ^ frame_round_idx[0] ^ frame_active_mask[0] ^ frame_syndrome_bits[0];

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            shot_open <= 1'b0;
            prediction_valid <= 1'b0;
            prediction_bits <= {OBS_W{1'b0}};
            shots_processed <= {COUNT_W{1'b0}};
            logical_error_count <= {COUNT_W{1'b0}};
            reported_latency_total_scaled <= {COUNT_W{1'b0}};
            reported_latency_ingestion_scaled <= {COUNT_W{1'b0}};
            reported_latency_graph_update_scaled <= {COUNT_W{1'b0}};
            reported_latency_matching_scaled <= {COUNT_W{1'b0}};
            reported_latency_correction_scaled <= {COUNT_W{1'b0}};
        end else begin
            prediction_valid <= 1'b0;

            if (shot_start || frame_valid)
                shot_open <= 1'b1;

            if (shot_open && expected_prediction_valid && observable_valid) begin
                prediction_valid <= 1'b1;
                prediction_bits <= expected_prediction_bits;
                shots_processed <= shots_processed + {{(COUNT_W-1){1'b0}}, 1'b1};
                if (expected_prediction_bits != observable_bits)
                    logical_error_count <= logical_error_count + {{(COUNT_W-1){1'b0}}, 1'b1};
                reported_latency_total_scaled <= latency_total_scaled;
                reported_latency_ingestion_scaled <= latency_ingestion_scaled;
                reported_latency_graph_update_scaled <= latency_graph_update_scaled;
                reported_latency_matching_scaled <= latency_matching_scaled;
                reported_latency_correction_scaled <= latency_correction_scaled;
                shot_open <= 1'b0;
            end
        end
    end
endmodule
```

```gitignore
sim_decoder_interface.out
*.vcd
vectors/current/
```

```text
# Placeholder file for the vectors parent directory.
```

- [ ] **Step 4: Run test to verify it passes**

Run from the repo root:

`Push-Location decoder`

`$env:PYTHONPATH='src'`

`python -m pytest tests/test_verilog_vectors.py tests/test_verilog_cli.py -v`

`python -m decoder_benchmark.verilog_cli --distance 3 --error-rate 0.001 --shots 4 --rounds 3 --seed 1234 --output-dir ..\verilog_test\vectors\current`

`Pop-Location`

`iverilog -o verilog_test\sim_decoder_interface.out verilog_test\decoder_interface_model.v verilog_test\tb_decoder_interface.v`

`vvp verilog_test\sim_decoder_interface.out`

Expected: Python tests PASS, Verilog compile succeeds, and `vvp` prints both PASS lines from `tb_decoder_interface.v`

- [ ] **Step 5: Run malformed-bundle smoke check**

Run from the repo root:

`$original = Get-Content verilog_test\vectors\current\frame_map.txt`

`$broken = @($original)`

`$broken[0] = "0 0 0"`

`Set-Content verilog_test\vectors\current\frame_map.txt $broken`

`vvp verilog_test\sim_decoder_interface.out`

`Set-Content verilog_test\vectors\current\frame_map.txt $original`

Expected: the rerun prints `ERROR: active mask cannot be zero`

- [ ] **Step 6: Commit**

```bash
git add verilog_test/tb_decoder_interface.v verilog_test/decoder_interface_model.v verilog_test/.gitignore verilog_test/vectors/.gitkeep
git commit -m "feat: add verilog interface simulation harness"
```
