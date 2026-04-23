# Surface-Code Real-Time MWPM FPGA Simulation Implementation Plan

> Status: Historical implementation plan. The work described here may already
> be partially or fully implemented. Use the corresponding architecture page and
> current source tree as the source of truth.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a simulation-first Python benchmark pipeline for rotated planar surface-code MWPM decoding that reports logical error rate, host decode wall time, and explicit simulated latency in syndrome-round units.

**Architecture:** Create a small `decoder_benchmark` package with one module per pipeline boundary: Stim circuit generation, syndrome-frame sampling, PyMatching decode, latency accounting, report generation, sinter baseline comparison, and CLI orchestration. Keep PyMatching as the reference decoder while exposing streaming-shaped interfaces that can later map to FPGA pipeline stages.

**Tech Stack:** Python 3.11+, Stim, PyMatching, sinter, NumPy, pytest, standard-library CSV/JSON/argparse/pathlib/dataclasses.

---

## File Structure

- Create: `pyproject.toml`
  Package metadata, runtime dependencies, pytest configuration, and the `decoder-benchmark` CLI entry point.
- Create: `README.md`
  User-facing quick start, dependency delegation notes, and example benchmark command.
- Create: `src/decoder_benchmark/__init__.py`
  Public package exports.
- Create: `src/decoder_benchmark/models.py`
  Shared dataclasses for benchmark configuration, latency costs, latency breakdowns, and result rows.
- Create: `src/decoder_benchmark/circuit_factory.py`
  Stim rotated planar memory circuit creation and detector error model extraction.
- Create: `src/decoder_benchmark/syndrome.py`
  Detector sample collection and conversion between batch detector arrays and ordered syndrome frames.
- Create: `src/decoder_benchmark/decoder.py`
  PyMatching-backed MWPM wrapper with batch decode and logical-error counting helpers.
- Create: `src/decoder_benchmark/latency.py`
  Replaceable latency model with deterministic stage-level accounting.
- Create: `src/decoder_benchmark/reporting.py`
  CSV, JSON, and Markdown benchmark bundle generation.
- Create: `src/decoder_benchmark/runner.py`
  End-to-end sweep runner that wires circuit generation, sampling, decoding, latency, and reporting.
- Create: `src/decoder_benchmark/sinter_baseline.py`
  Tiny sinter baseline collection path and conversion to comparable metrics.
- Create: `src/decoder_benchmark/cli.py`
  Command-line interface for benchmark sweeps and output directory selection.
- Create: `tests/test_circuit_factory.py`
  Stim circuit and detector error model tests.
- Create: `tests/test_syndrome.py`
  Streaming frame conversion tests.
- Create: `tests/test_decoder.py`
  PyMatching correctness path tests.
- Create: `tests/test_latency.py`
  Deterministic latency accounting tests.
- Create: `tests/test_reporting.py`
  Artifact generation tests.
- Create: `tests/test_runner.py`
  End-to-end tiny benchmark tests.
- Create: `tests/test_sinter_baseline.py`
  Sinter baseline conversion tests.
- Create: `tests/test_cli.py`
  CLI argument parsing and smoke execution tests.

## Task 1: Project Scaffold

**Files:**
- Create: `pyproject.toml`
- Create: `README.md`
- Create: `src/decoder_benchmark/__init__.py`
- Create: `tests/test_package_import.py`

- [ ] **Step 1: Write the failing package import test**

Create `tests/test_package_import.py` with:

```python
import decoder_benchmark


def test_package_exposes_version():
    assert isinstance(decoder_benchmark.__version__, str)
    assert decoder_benchmark.__version__
```

- [ ] **Step 2: Run the import test to verify it fails**

Run:

```bash
python -m pytest tests/test_package_import.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark'`.

- [ ] **Step 3: Create package metadata**

Create `pyproject.toml` with:

```toml
[build-system]
requires = ["hatchling>=1.24"]
build-backend = "hatchling.build"

[project]
name = "decoder-benchmark"
version = "0.1.0"
description = "Simulation-first rotated surface-code MWPM benchmark with explicit latency modeling."
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
  "numpy>=1.26",
  "stim>=1.13",
  "pymatching>=2.2",
  "sinter>=1.13",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.0",
]

[project.scripts]
decoder-benchmark = "decoder_benchmark.cli:main"

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
addopts = "-ra"
```

- [ ] **Step 4: Create initial package export**

Create `src/decoder_benchmark/__init__.py` with:

```python
"""Rotated surface-code MWPM benchmark pipeline."""

__version__ = "0.1.0"
```

- [ ] **Step 5: Create quick-start documentation**

Create `README.md` with:

```markdown
# decoder-benchmark

Simulation-first benchmark tooling for rotated planar surface-code MWPM decoding.

The milestone-one implementation delegates circuit generation and detector sampling
to Stim, reference MWPM decoding to PyMatching, and a tiny baseline cross-check to
sinter. Custom code in this repository owns the streaming-shaped syndrome frame
representation, latency accounting, benchmark orchestration, and report packaging.

## Development setup

```bash
python -m pip install -e ".[dev]"
python -m pytest -v
```

## Tiny development run

```bash
decoder-benchmark \
  --distances 3 \
  --error-rates 0.001 \
  --shots 100 \
  --output-dir benchmark-out/dev
```

The command writes `results.csv`, `results.json`, and `report.md` into the output
directory.

## Milestone boundary

This project does not implement RTL, HLS, FPGA board integration, or a custom MWPM
engine. The latency numbers are explicit simulation parameters around a reference
software decoder; they are design aids, not hardware timing-closure evidence.
```

- [ ] **Step 6: Install package in editable mode**

Run:

```bash
python -m pip install -e ".[dev]"
```

Expected: installation completes and includes `decoder-benchmark`.

- [ ] **Step 7: Run the import test to verify it passes**

Run:

```bash
python -m pytest tests/test_package_import.py -v
```

Expected: PASS.

- [ ] **Step 8: Commit scaffold**

Run:

```bash
git add pyproject.toml README.md src/decoder_benchmark/__init__.py tests/test_package_import.py
git commit -m "chore: scaffold decoder benchmark package"
```

## Task 2: Shared Models

**Files:**
- Create: `src/decoder_benchmark/models.py`
- Modify: `src/decoder_benchmark/__init__.py`
- Create: `tests/test_models.py`

- [ ] **Step 1: Write failing model tests**

Create `tests/test_models.py` with:

```python
from decoder_benchmark.models import (
    BenchmarkConfig,
    BenchmarkResult,
    LatencyBreakdown,
    LatencyStageCosts,
)


def test_benchmark_config_defaults_rounds_to_distance():
    config = BenchmarkConfig(distance=5, physical_error_rate=0.003, shots=100)

    assert config.rounds == 5
    assert config.basis == "z"


def test_latency_stage_costs_have_named_stage_dict():
    costs = LatencyStageCosts(
        ingestion=0.25,
        graph_update=0.5,
        matching=1.25,
        correction_output=0.25,
    )

    assert costs.as_dict() == {
        "ingestion": 0.25,
        "graph_update": 0.5,
        "matching": 1.25,
        "correction_output": 0.25,
    }


def test_benchmark_result_serializes_flat_row_and_structured_dict():
    result = BenchmarkResult(
        distance=3,
        physical_error_rate=0.001,
        rounds=3,
        shots=10,
        logical_errors=1,
        logical_error_rate=0.1,
        decode_wall_time_seconds=0.012,
        latency=LatencyBreakdown(
            latency_rounds=2.5,
            stage_breakdown={
                "ingestion": 0.75,
                "graph_update": 0.75,
                "matching": 0.75,
                "correction_output": 0.25,
            },
        ),
    )

    assert result.to_csv_row() == {
        "distance": 3,
        "physical_error_rate": 0.001,
        "rounds": 3,
        "shots": 10,
        "logical_errors": 1,
        "logical_error_rate": 0.1,
        "decode_wall_time_seconds": 0.012,
        "latency_rounds": 2.5,
        "latency_ingestion_rounds": 0.75,
        "latency_graph_update_rounds": 0.75,
        "latency_matching_rounds": 0.75,
        "latency_correction_output_rounds": 0.25,
    }
    assert result.to_json_dict()["latency"]["stage_breakdown"]["matching"] == 0.75
```

- [ ] **Step 2: Run model tests to verify they fail**

Run:

```bash
python -m pytest tests/test_models.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.models'`.

- [ ] **Step 3: Implement shared models**

Create `src/decoder_benchmark/models.py` with:

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Any


@dataclass(frozen=True)
class BenchmarkConfig:
    distance: int
    physical_error_rate: float
    shots: int
    rounds: int | None = None
    basis: str = "z"

    def __post_init__(self) -> None:
        if self.distance < 3 or self.distance % 2 == 0:
            raise ValueError("distance must be an odd integer greater than or equal to 3")
        if not 0.0 < self.physical_error_rate < 1.0:
            raise ValueError("physical_error_rate must be between 0 and 1")
        if self.shots < 1:
            raise ValueError("shots must be at least 1")
        if self.rounds is None:
            object.__setattr__(self, "rounds", self.distance)
        if self.rounds is None or self.rounds < 1:
            raise ValueError("rounds must be at least 1")
        normalized_basis = self.basis.lower()
        if normalized_basis not in {"x", "z"}:
            raise ValueError("basis must be 'x' or 'z'")
        object.__setattr__(self, "basis", normalized_basis)


@dataclass(frozen=True)
class LatencyStageCosts:
    ingestion: float = 0.25
    graph_update: float = 0.25
    matching: float = 1.0
    correction_output: float = 0.25

    def __post_init__(self) -> None:
        for name, value in self.as_dict().items():
            if value < 0:
                raise ValueError(f"{name} latency cost must be non-negative")

    def as_dict(self) -> dict[str, float]:
        return {
            "ingestion": self.ingestion,
            "graph_update": self.graph_update,
            "matching": self.matching,
            "correction_output": self.correction_output,
        }


@dataclass(frozen=True)
class LatencyBreakdown:
    latency_rounds: float
    stage_breakdown: dict[str, float]

    def to_json_dict(self) -> dict[str, Any]:
        return {
            "latency_rounds": self.latency_rounds,
            "stage_breakdown": dict(self.stage_breakdown),
        }


@dataclass(frozen=True)
class BenchmarkResult:
    distance: int
    physical_error_rate: float
    rounds: int
    shots: int
    logical_errors: int
    logical_error_rate: float
    decode_wall_time_seconds: float
    latency: LatencyBreakdown

    def to_csv_row(self) -> dict[str, int | float]:
        return {
            "distance": self.distance,
            "physical_error_rate": self.physical_error_rate,
            "rounds": self.rounds,
            "shots": self.shots,
            "logical_errors": self.logical_errors,
            "logical_error_rate": self.logical_error_rate,
            "decode_wall_time_seconds": self.decode_wall_time_seconds,
            "latency_rounds": self.latency.latency_rounds,
            "latency_ingestion_rounds": self.latency.stage_breakdown["ingestion"],
            "latency_graph_update_rounds": self.latency.stage_breakdown["graph_update"],
            "latency_matching_rounds": self.latency.stage_breakdown["matching"],
            "latency_correction_output_rounds": self.latency.stage_breakdown["correction_output"],
        }

    def to_json_dict(self) -> dict[str, Any]:
        return {
            "distance": self.distance,
            "physical_error_rate": self.physical_error_rate,
            "rounds": self.rounds,
            "shots": self.shots,
            "logical_errors": self.logical_errors,
            "logical_error_rate": self.logical_error_rate,
            "decode_wall_time_seconds": self.decode_wall_time_seconds,
            "latency": self.latency.to_json_dict(),
        }
```

- [ ] **Step 4: Export model names**

Replace `src/decoder_benchmark/__init__.py` with:

```python
"""Rotated surface-code MWPM benchmark pipeline."""

from decoder_benchmark.models import (
    BenchmarkConfig,
    BenchmarkResult,
    LatencyBreakdown,
    LatencyStageCosts,
)

__version__ = "0.1.0"

__all__ = [
    "BenchmarkConfig",
    "BenchmarkResult",
    "LatencyBreakdown",
    "LatencyStageCosts",
    "__version__",
]
```

- [ ] **Step 5: Run model and import tests**

Run:

```bash
python -m pytest tests/test_models.py tests/test_package_import.py -v
```

Expected: PASS.

- [ ] **Step 6: Commit shared models**

Run:

```bash
git add src/decoder_benchmark/__init__.py src/decoder_benchmark/models.py tests/test_models.py
git commit -m "feat: add benchmark data models"
```

## Task 3: CircuitFactory

**Files:**
- Create: `src/decoder_benchmark/circuit_factory.py`
- Create: `tests/test_circuit_factory.py`

- [ ] **Step 1: Write failing circuit factory tests**

Create `tests/test_circuit_factory.py` with:

```python
import stim

from decoder_benchmark.circuit_factory import CircuitFactory
from decoder_benchmark.models import BenchmarkConfig


def test_builds_rotated_memory_circuit_with_observable_and_detectors():
    factory = CircuitFactory()
    config = BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=5)

    circuit = factory.build_memory_circuit(config)

    assert isinstance(circuit, stim.Circuit)
    assert circuit.num_detectors > 0
    assert circuit.num_observables == 1


def test_extracts_detector_error_model_for_matching():
    factory = CircuitFactory()
    config = BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=5)
    circuit = factory.build_memory_circuit(config)

    detector_error_model = factory.detector_error_model(circuit)

    assert isinstance(detector_error_model, stim.DetectorErrorModel)
    assert len(str(detector_error_model)) > 0


def test_supports_x_and_z_memory_basis():
    factory = CircuitFactory()
    z_circuit = factory.build_memory_circuit(
        BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=5, basis="z")
    )
    x_circuit = factory.build_memory_circuit(
        BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=5, basis="x")
    )

    assert z_circuit.num_detectors == x_circuit.num_detectors
    assert z_circuit.num_observables == x_circuit.num_observables == 1
```

- [ ] **Step 2: Run circuit tests to verify they fail**

Run:

```bash
python -m pytest tests/test_circuit_factory.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.circuit_factory'`.

- [ ] **Step 3: Implement CircuitFactory**

Create `src/decoder_benchmark/circuit_factory.py` with:

```python
from __future__ import annotations

import stim

from decoder_benchmark.models import BenchmarkConfig


class CircuitFactory:
    """Creates Stim rotated planar surface-code memory circuits."""

    def build_memory_circuit(self, config: BenchmarkConfig) -> stim.Circuit:
        task_name = f"surface_code:rotated_memory_{config.basis}"
        return stim.Circuit.generated(
            task_name,
            distance=config.distance,
            rounds=config.rounds,
            after_clifford_depolarization=config.physical_error_rate,
            before_round_data_depolarization=config.physical_error_rate,
            before_measure_flip_probability=config.physical_error_rate,
            after_reset_flip_probability=config.physical_error_rate,
        )

    def detector_error_model(self, circuit: stim.Circuit) -> stim.DetectorErrorModel:
        return circuit.detector_error_model(decompose_errors=True)
```

- [ ] **Step 4: Run circuit tests**

Run:

```bash
python -m pytest tests/test_circuit_factory.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit CircuitFactory**

Run:

```bash
git add src/decoder_benchmark/circuit_factory.py tests/test_circuit_factory.py
git commit -m "feat: add stim circuit factory"
```

## Task 4: SyndromeSampler

**Files:**
- Create: `src/decoder_benchmark/syndrome.py`
- Create: `tests/test_syndrome.py`

- [ ] **Step 1: Write failing syndrome sampler tests**

Create `tests/test_syndrome.py` with:

```python
import numpy as np

from decoder_benchmark.circuit_factory import CircuitFactory
from decoder_benchmark.models import BenchmarkConfig
from decoder_benchmark.syndrome import SyndromeFrame, SyndromeSampler


def test_samples_detector_and_observable_arrays():
    circuit = CircuitFactory().build_memory_circuit(
        BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=4)
    )
    sampler = SyndromeSampler(seed=1234)

    detector_samples, observable_samples = sampler.sample_batch(circuit, shots=4)

    assert detector_samples.shape == (4, circuit.num_detectors)
    assert observable_samples.shape == (4, circuit.num_observables)
    assert detector_samples.dtype == np.bool_
    assert observable_samples.dtype == np.bool_


def test_converts_detector_batch_to_ordered_frames_without_losing_bits():
    circuit = CircuitFactory().build_memory_circuit(
        BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=2)
    )
    sampler = SyndromeSampler(seed=1234)
    detector_samples, _ = sampler.sample_batch(circuit, shots=2)

    frames_by_shot = sampler.to_frames(circuit, detector_samples)
    reconstructed = sampler.frames_to_batch(frames_by_shot, detector_count=circuit.num_detectors)

    assert len(frames_by_shot) == 2
    assert all(isinstance(frame, SyndromeFrame) for frame in frames_by_shot[0])
    assert np.array_equal(reconstructed, detector_samples)


def test_detector_round_groups_are_sorted_and_cover_all_detectors():
    circuit = CircuitFactory().build_memory_circuit(
        BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=1)
    )
    sampler = SyndromeSampler(seed=1234)

    groups = sampler.detector_round_groups(circuit)
    flat_detector_ids = [detector_id for group in groups for detector_id in group.detector_indices]

    assert groups == sorted(groups, key=lambda group: group.round_index)
    assert flat_detector_ids == sorted(flat_detector_ids)
    assert flat_detector_ids == list(range(circuit.num_detectors))
```

- [ ] **Step 2: Run syndrome tests to verify they fail**

Run:

```bash
python -m pytest tests/test_syndrome.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.syndrome'`.

- [ ] **Step 3: Implement SyndromeSampler**

Create `src/decoder_benchmark/syndrome.py` with:

```python
from __future__ import annotations

from dataclasses import dataclass

import numpy as np
import stim


@dataclass(frozen=True)
class DetectorRoundGroup:
    round_index: int
    detector_indices: tuple[int, ...]


@dataclass(frozen=True)
class SyndromeFrame:
    round_index: int
    detector_indices: tuple[int, ...]
    values: np.ndarray


class SyndromeSampler:
    """Samples Stim detector data and exposes it as time-ordered frames."""

    def __init__(self, seed: int | None = None) -> None:
        self.seed = seed

    def sample_batch(self, circuit: stim.Circuit, shots: int) -> tuple[np.ndarray, np.ndarray]:
        sampler = circuit.compile_detector_sampler(seed=self.seed)
        detector_samples, observable_samples = sampler.sample(
            shots=shots,
            separate_observables=True,
        )
        return np.asarray(detector_samples, dtype=np.bool_), np.asarray(observable_samples, dtype=np.bool_)

    def detector_round_groups(self, circuit: stim.Circuit) -> list[DetectorRoundGroup]:
        detector_coordinates = circuit.get_detector_coordinates()
        grouped: dict[int, list[int]] = {}
        for detector_id in range(circuit.num_detectors):
            coordinates = detector_coordinates.get(detector_id, ())
            round_index = int(round(coordinates[-1])) if coordinates else 0
            grouped.setdefault(round_index, []).append(detector_id)

        return [
            DetectorRoundGroup(round_index=round_index, detector_indices=tuple(sorted(detector_ids)))
            for round_index, detector_ids in sorted(grouped.items())
        ]

    def to_frames(self, circuit: stim.Circuit, detector_samples: np.ndarray) -> list[list[SyndromeFrame]]:
        samples = np.asarray(detector_samples, dtype=np.bool_)
        if samples.ndim != 2:
            raise ValueError("detector_samples must be a 2D array shaped as shots by detectors")
        if samples.shape[1] != circuit.num_detectors:
            raise ValueError("detector_samples detector dimension must match circuit.num_detectors")

        groups = self.detector_round_groups(circuit)
        frames_by_shot: list[list[SyndromeFrame]] = []
        for shot in samples:
            frames_by_shot.append(
                [
                    SyndromeFrame(
                        round_index=group.round_index,
                        detector_indices=group.detector_indices,
                        values=np.asarray(shot[list(group.detector_indices)], dtype=np.bool_),
                    )
                    for group in groups
                ]
            )
        return frames_by_shot

    def frames_to_batch(self, frames_by_shot: list[list[SyndromeFrame]], detector_count: int) -> np.ndarray:
        reconstructed = np.zeros((len(frames_by_shot), detector_count), dtype=np.bool_)
        for shot_index, frames in enumerate(frames_by_shot):
            for frame in frames:
                reconstructed[shot_index, list(frame.detector_indices)] = frame.values
        return reconstructed
```

- [ ] **Step 4: Run syndrome tests**

Run:

```bash
python -m pytest tests/test_syndrome.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit SyndromeSampler**

Run:

```bash
git add src/decoder_benchmark/syndrome.py tests/test_syndrome.py
git commit -m "feat: add syndrome frame sampler"
```

## Task 5: MWPMDecoder

**Files:**
- Create: `src/decoder_benchmark/decoder.py`
- Create: `tests/test_decoder.py`

- [ ] **Step 1: Write failing decoder tests**

Create `tests/test_decoder.py` with:

```python
import numpy as np

from decoder_benchmark.circuit_factory import CircuitFactory
from decoder_benchmark.decoder import MWPMDecoder, count_logical_errors
from decoder_benchmark.models import BenchmarkConfig
from decoder_benchmark.syndrome import SyndromeSampler


def test_builds_decoder_from_detector_error_model_and_decodes_batch():
    factory = CircuitFactory()
    config = BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=8)
    circuit = factory.build_memory_circuit(config)
    detector_error_model = factory.detector_error_model(circuit)
    detector_samples, observable_samples = SyndromeSampler(seed=1234).sample_batch(circuit, shots=8)

    decoder = MWPMDecoder.from_detector_error_model(detector_error_model)
    predictions = decoder.decode_batch(detector_samples)

    assert predictions.shape == observable_samples.shape
    assert predictions.dtype == np.bool_


def test_counts_logical_errors_from_prediction_mismatch():
    predictions = np.array([[False], [True], [True], [False]], dtype=np.bool_)
    observables = np.array([[False], [False], [True], [True]], dtype=np.bool_)

    logical_errors, logical_error_rate = count_logical_errors(predictions, observables)

    assert logical_errors == 2
    assert logical_error_rate == 0.5
```

- [ ] **Step 2: Run decoder tests to verify they fail**

Run:

```bash
python -m pytest tests/test_decoder.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.decoder'`.

- [ ] **Step 3: Implement MWPMDecoder**

Create `src/decoder_benchmark/decoder.py` with:

```python
from __future__ import annotations

import numpy as np
import pymatching
import stim


class MWPMDecoder:
    """PyMatching-backed reference MWPM decoder with a streaming-facing boundary."""

    def __init__(self, matching: pymatching.Matching) -> None:
        self.matching = matching

    @classmethod
    def from_detector_error_model(cls, detector_error_model: stim.DetectorErrorModel) -> "MWPMDecoder":
        return cls(pymatching.Matching.from_detector_error_model(detector_error_model))

    def decode_batch(self, detector_samples: np.ndarray) -> np.ndarray:
        samples = np.asarray(detector_samples, dtype=np.bool_)
        if samples.ndim != 2:
            raise ValueError("detector_samples must be a 2D array shaped as shots by detectors")

        predictions = np.asarray(self.matching.decode_batch(samples), dtype=np.bool_)
        if predictions.ndim == 1:
            predictions = predictions.reshape((-1, 1))
        return predictions


def count_logical_errors(predictions: np.ndarray, observables: np.ndarray) -> tuple[int, float]:
    prediction_array = np.asarray(predictions, dtype=np.bool_)
    observable_array = np.asarray(observables, dtype=np.bool_)
    if prediction_array.shape != observable_array.shape:
        raise ValueError("predictions and observables must have the same shape")

    mismatched_shots = np.any(prediction_array != observable_array, axis=1)
    logical_errors = int(np.count_nonzero(mismatched_shots))
    return logical_errors, logical_errors / prediction_array.shape[0]
```

- [ ] **Step 4: Run decoder tests**

Run:

```bash
python -m pytest tests/test_decoder.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit MWPMDecoder**

Run:

```bash
git add src/decoder_benchmark/decoder.py tests/test_decoder.py
git commit -m "feat: add pymatching decoder wrapper"
```

## Task 6: LatencyModel

**Files:**
- Create: `src/decoder_benchmark/latency.py`
- Create: `tests/test_latency.py`

- [ ] **Step 1: Write failing latency tests**

Create `tests/test_latency.py` with:

```python
from decoder_benchmark.latency import LatencyModel
from decoder_benchmark.models import LatencyStageCosts


def test_latency_model_accounts_for_per_frame_and_final_stages():
    model = LatencyModel(
        LatencyStageCosts(
            ingestion=0.25,
            graph_update=0.5,
            matching=1.25,
            correction_output=0.25,
        )
    )

    breakdown = model.evaluate(frame_count=3)

    assert breakdown.latency_rounds == 3.5
    assert breakdown.stage_breakdown == {
        "ingestion": 0.75,
        "graph_update": 1.5,
        "matching": 1.25,
        "correction_output": 0.25,
    }


def test_latency_model_rejects_empty_stream():
    model = LatencyModel()

    try:
        model.evaluate(frame_count=0)
    except ValueError as exc:
        assert str(exc) == "frame_count must be at least 1"
    else:
        raise AssertionError("expected ValueError")
```

- [ ] **Step 2: Run latency tests to verify they fail**

Run:

```bash
python -m pytest tests/test_latency.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.latency'`.

- [ ] **Step 3: Implement LatencyModel**

Create `src/decoder_benchmark/latency.py` with:

```python
from __future__ import annotations

from decoder_benchmark.models import LatencyBreakdown, LatencyStageCosts


class LatencyModel:
    """Deterministic syndrome-round latency accounting for a simple decoder pipeline."""

    def __init__(self, costs: LatencyStageCosts | None = None) -> None:
        self.costs = costs or LatencyStageCosts()

    def evaluate(self, frame_count: int) -> LatencyBreakdown:
        if frame_count < 1:
            raise ValueError("frame_count must be at least 1")

        stage_breakdown = {
            "ingestion": self.costs.ingestion * frame_count,
            "graph_update": self.costs.graph_update * frame_count,
            "matching": self.costs.matching,
            "correction_output": self.costs.correction_output,
        }
        latency_rounds = sum(stage_breakdown.values())
        return LatencyBreakdown(latency_rounds=latency_rounds, stage_breakdown=stage_breakdown)
```

- [ ] **Step 4: Run latency tests**

Run:

```bash
python -m pytest tests/test_latency.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit LatencyModel**

Run:

```bash
git add src/decoder_benchmark/latency.py tests/test_latency.py
git commit -m "feat: add deterministic latency model"
```

## Task 7: ReportGenerator

**Files:**
- Create: `src/decoder_benchmark/reporting.py`
- Create: `tests/test_reporting.py`

- [ ] **Step 1: Write failing report generation tests**

Create `tests/test_reporting.py` with:

```python
import csv
import json

from decoder_benchmark.models import BenchmarkResult, LatencyBreakdown
from decoder_benchmark.reporting import ReportGenerator


def sample_result() -> BenchmarkResult:
    return BenchmarkResult(
        distance=3,
        physical_error_rate=0.001,
        rounds=3,
        shots=10,
        logical_errors=1,
        logical_error_rate=0.1,
        decode_wall_time_seconds=0.012,
        latency=LatencyBreakdown(
            latency_rounds=2.5,
            stage_breakdown={
                "ingestion": 0.75,
                "graph_update": 0.75,
                "matching": 0.75,
                "correction_output": 0.25,
            },
        ),
    )


def test_writes_csv_json_and_markdown_bundle(tmp_path):
    generator = ReportGenerator(tmp_path)

    paths = generator.write(
        results=[sample_result()],
        metadata={
            "distances": [3],
            "physical_error_rates": [0.001],
            "shots": 10,
            "rounds": "distance",
        },
    )

    assert paths.csv_path == tmp_path / "results.csv"
    assert paths.json_path == tmp_path / "results.json"
    assert paths.markdown_path == tmp_path / "report.md"
    assert paths.csv_path.exists()
    assert paths.json_path.exists()
    assert paths.markdown_path.exists()

    with paths.csv_path.open(newline="") as csv_file:
        rows = list(csv.DictReader(csv_file))
    assert rows[0]["distance"] == "3"
    assert rows[0]["latency_rounds"] == "2.5"

    payload = json.loads(paths.json_path.read_text())
    assert payload["metadata"]["shots"] == 10
    assert payload["results"][0]["latency"]["stage_breakdown"]["matching"] == 0.75

    markdown = paths.markdown_path.read_text()
    assert "| distance | p_error | rounds | shots | logical_error_rate | latency_rounds |" in markdown
    assert "| 3 | 0.001 | 3 | 10 | 0.1 | 2.5 |" in markdown
```

- [ ] **Step 2: Run reporting tests to verify they fail**

Run:

```bash
python -m pytest tests/test_reporting.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.reporting'`.

- [ ] **Step 3: Implement ReportGenerator**

Create `src/decoder_benchmark/reporting.py` with:

```python
from __future__ import annotations

import csv
import json
from dataclasses import dataclass
from pathlib import Path
from typing import Any

from decoder_benchmark.models import BenchmarkResult


@dataclass(frozen=True)
class ReportPaths:
    csv_path: Path
    json_path: Path
    markdown_path: Path


class ReportGenerator:
    def __init__(self, output_dir: str | Path) -> None:
        self.output_dir = Path(output_dir)

    def write(self, results: list[BenchmarkResult], metadata: dict[str, Any]) -> ReportPaths:
        self.output_dir.mkdir(parents=True, exist_ok=True)
        csv_path = self.output_dir / "results.csv"
        json_path = self.output_dir / "results.json"
        markdown_path = self.output_dir / "report.md"

        self._write_csv(csv_path, results)
        self._write_json(json_path, results, metadata)
        self._write_markdown(markdown_path, results, metadata)
        return ReportPaths(csv_path=csv_path, json_path=json_path, markdown_path=markdown_path)

    def _write_csv(self, path: Path, results: list[BenchmarkResult]) -> None:
        fieldnames = [
            "distance",
            "physical_error_rate",
            "rounds",
            "shots",
            "logical_errors",
            "logical_error_rate",
            "decode_wall_time_seconds",
            "latency_rounds",
            "latency_ingestion_rounds",
            "latency_graph_update_rounds",
            "latency_matching_rounds",
            "latency_correction_output_rounds",
        ]
        with path.open("w", newline="") as csv_file:
            writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
            writer.writeheader()
            for result in results:
                writer.writerow(result.to_csv_row())

    def _write_json(self, path: Path, results: list[BenchmarkResult], metadata: dict[str, Any]) -> None:
        payload = {
            "metadata": metadata,
            "results": [result.to_json_dict() for result in results],
        }
        path.write_text(json.dumps(payload, indent=2, sort_keys=True) + "\n")

    def _write_markdown(self, path: Path, results: list[BenchmarkResult], metadata: dict[str, Any]) -> None:
        lines = [
            "# Surface-Code MWPM Benchmark Report",
            "",
            "## Run Metadata",
            "",
        ]
        for key, value in metadata.items():
            lines.append(f"- `{key}`: `{value}`")
        lines.extend(
            [
                "",
                "## Results",
                "",
                "| distance | p_error | rounds | shots | logical_error_rate | latency_rounds |",
                "| --- | --- | --- | --- | --- | --- |",
            ]
        )
        for result in results:
            lines.append(
                "| "
                f"{result.distance} | "
                f"{result.physical_error_rate} | "
                f"{result.rounds} | "
                f"{result.shots} | "
                f"{result.logical_error_rate} | "
                f"{result.latency.latency_rounds} |"
            )
        lines.extend(
            [
                "",
                "## Interpretation",
                "",
                "Logical error rate is measured by comparing PyMatching-predicted observable flips "
                "against Stim-sampled observables. Latency is a configurable syndrome-round model "
                "with explicit ingestion, graph update, matching, and correction output stages.",
            ]
        )
        path.write_text("\n".join(lines) + "\n")
```

- [ ] **Step 4: Run reporting tests**

Run:

```bash
python -m pytest tests/test_reporting.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit ReportGenerator**

Run:

```bash
git add src/decoder_benchmark/reporting.py tests/test_reporting.py
git commit -m "feat: add benchmark report generation"
```

## Task 8: BenchmarkRunner

**Files:**
- Create: `src/decoder_benchmark/runner.py`
- Create: `tests/test_runner.py`

- [ ] **Step 1: Write failing runner tests**

Create `tests/test_runner.py` with:

```python
from decoder_benchmark.models import BenchmarkConfig, LatencyStageCosts
from decoder_benchmark.runner import BenchmarkRunner


def test_runs_tiny_single_configuration():
    runner = BenchmarkRunner(
        latency_costs=LatencyStageCosts(
            ingestion=0.25,
            graph_update=0.25,
            matching=0.5,
            correction_output=0.25,
        ),
        seed=1234,
    )

    result = runner.run_config(
        BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=8, rounds=3)
    )

    assert result.distance == 3
    assert result.physical_error_rate == 0.001
    assert result.rounds == 3
    assert result.shots == 8
    assert 0 <= result.logical_errors <= 8
    assert 0.0 <= result.logical_error_rate <= 1.0
    assert result.decode_wall_time_seconds >= 0.0
    assert result.latency.latency_rounds > 0.0


def test_runs_sweep_with_rounds_defaulting_to_distance():
    runner = BenchmarkRunner(seed=1234)

    results = runner.run_sweep(distances=[3], physical_error_rates=[0.001, 0.003], shots=4)

    assert [result.physical_error_rate for result in results] == [0.001, 0.003]
    assert all(result.rounds == result.distance for result in results)
```

- [ ] **Step 2: Run runner tests to verify they fail**

Run:

```bash
python -m pytest tests/test_runner.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.runner'`.

- [ ] **Step 3: Implement BenchmarkRunner**

Create `src/decoder_benchmark/runner.py` with:

```python
from __future__ import annotations

import time

from decoder_benchmark.circuit_factory import CircuitFactory
from decoder_benchmark.decoder import MWPMDecoder, count_logical_errors
from decoder_benchmark.latency import LatencyModel
from decoder_benchmark.models import BenchmarkConfig, BenchmarkResult, LatencyStageCosts
from decoder_benchmark.syndrome import SyndromeSampler


class BenchmarkRunner:
    def __init__(
        self,
        circuit_factory: CircuitFactory | None = None,
        latency_costs: LatencyStageCosts | None = None,
        seed: int | None = None,
    ) -> None:
        self.circuit_factory = circuit_factory or CircuitFactory()
        self.latency_model = LatencyModel(latency_costs)
        self.sampler = SyndromeSampler(seed=seed)

    def run_config(self, config: BenchmarkConfig) -> BenchmarkResult:
        circuit = self.circuit_factory.build_memory_circuit(config)
        detector_error_model = self.circuit_factory.detector_error_model(circuit)
        detector_samples, observable_samples = self.sampler.sample_batch(circuit, config.shots)
        frames_by_shot = self.sampler.to_frames(circuit, detector_samples)

        decoder = MWPMDecoder.from_detector_error_model(detector_error_model)
        started_at = time.perf_counter()
        predictions = decoder.decode_batch(detector_samples)
        decode_wall_time_seconds = time.perf_counter() - started_at

        logical_errors, logical_error_rate = count_logical_errors(predictions, observable_samples)
        frame_count = len(frames_by_shot[0]) if frames_by_shot else 0
        latency = self.latency_model.evaluate(frame_count=frame_count)

        return BenchmarkResult(
            distance=config.distance,
            physical_error_rate=config.physical_error_rate,
            rounds=config.rounds or config.distance,
            shots=config.shots,
            logical_errors=logical_errors,
            logical_error_rate=logical_error_rate,
            decode_wall_time_seconds=decode_wall_time_seconds,
            latency=latency,
        )

    def run_sweep(
        self,
        distances: list[int],
        physical_error_rates: list[float],
        shots: int,
        rounds: int | None = None,
        basis: str = "z",
    ) -> list[BenchmarkResult]:
        results: list[BenchmarkResult] = []
        for distance in distances:
            for physical_error_rate in physical_error_rates:
                config = BenchmarkConfig(
                    distance=distance,
                    physical_error_rate=physical_error_rate,
                    shots=shots,
                    rounds=rounds,
                    basis=basis,
                )
                results.append(self.run_config(config))
        return results
```

- [ ] **Step 4: Run runner tests**

Run:

```bash
python -m pytest tests/test_runner.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit BenchmarkRunner**

Run:

```bash
git add src/decoder_benchmark/runner.py tests/test_runner.py
git commit -m "feat: add benchmark sweep runner"
```

## Task 9: Sinter Baseline Cross-Check

**Files:**
- Create: `src/decoder_benchmark/sinter_baseline.py`
- Create: `tests/test_sinter_baseline.py`

- [ ] **Step 1: Write failing sinter baseline tests**

Create `tests/test_sinter_baseline.py` with:

```python
from decoder_benchmark.models import BenchmarkConfig
from decoder_benchmark.sinter_baseline import SinterBaselineResult, run_sinter_baseline


def test_sinter_baseline_returns_comparable_error_metrics():
    result = run_sinter_baseline(
        BenchmarkConfig(distance=3, physical_error_rate=0.001, shots=4, rounds=3),
        max_shots=4,
        num_workers=1,
    )

    assert isinstance(result, SinterBaselineResult)
    assert result.shots == 4
    assert 0 <= result.errors <= result.shots
    assert 0.0 <= result.logical_error_rate <= 1.0
```

- [ ] **Step 2: Run sinter baseline tests to verify they fail**

Run:

```bash
python -m pytest tests/test_sinter_baseline.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.sinter_baseline'`.

- [ ] **Step 3: Implement sinter baseline helper**

Create `src/decoder_benchmark/sinter_baseline.py` with:

```python
from __future__ import annotations

from dataclasses import dataclass

import sinter

from decoder_benchmark.circuit_factory import CircuitFactory
from decoder_benchmark.models import BenchmarkConfig


@dataclass(frozen=True)
class SinterBaselineResult:
    shots: int
    errors: int
    logical_error_rate: float
    seconds: float


def run_sinter_baseline(
    config: BenchmarkConfig,
    max_shots: int | None = None,
    num_workers: int = 1,
) -> SinterBaselineResult:
    circuit = CircuitFactory().build_memory_circuit(config)
    task = sinter.Task(
        circuit=circuit,
        decoder="pymatching",
        json_metadata={
            "distance": config.distance,
            "physical_error_rate": config.physical_error_rate,
            "rounds": config.rounds,
            "basis": config.basis,
        },
    )
    stats = sinter.collect(
        num_workers=num_workers,
        tasks=[task],
        max_shots=max_shots or config.shots,
        max_errors=max_shots or config.shots,
    )
    if len(stats) != 1:
        raise RuntimeError(f"expected one sinter result, got {len(stats)}")
    stat = stats[0]
    logical_error_rate = stat.errors / stat.shots if stat.shots else 0.0
    return SinterBaselineResult(
        shots=stat.shots,
        errors=stat.errors,
        logical_error_rate=logical_error_rate,
        seconds=stat.seconds,
    )
```

- [ ] **Step 4: Run sinter baseline tests**

Run:

```bash
python -m pytest tests/test_sinter_baseline.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit sinter baseline helper**

Run:

```bash
git add src/decoder_benchmark/sinter_baseline.py tests/test_sinter_baseline.py
git commit -m "feat: add sinter baseline cross-check"
```

## Task 10: CLI And Benchmark Bundle

**Files:**
- Create: `src/decoder_benchmark/cli.py`
- Create: `tests/test_cli.py`
- Modify: `README.md`

- [ ] **Step 1: Write failing CLI tests**

Create `tests/test_cli.py` with:

```python
import json

from decoder_benchmark.cli import main, parse_args


def test_parse_args_supports_default_milestone_sweep():
    args = parse_args(
        [
            "--shots",
            "10",
            "--output-dir",
            "benchmark-out/test",
        ]
    )

    assert args.distances == [3, 5, 7]
    assert args.error_rates == [0.001, 0.003, 0.01]
    assert args.shots == 10
    assert args.rounds is None
    assert args.output_dir == "benchmark-out/test"


def test_main_writes_benchmark_bundle(tmp_path):
    exit_code = main(
        [
            "--distances",
            "3",
            "--error-rates",
            "0.001",
            "--shots",
            "4",
            "--rounds",
            "3",
            "--output-dir",
            str(tmp_path),
            "--seed",
            "1234",
            "--ingestion-rounds",
            "0.25",
            "--graph-update-rounds",
            "0.25",
            "--matching-rounds",
            "0.5",
            "--correction-output-rounds",
            "0.25",
        ]
    )

    assert exit_code == 0
    assert (tmp_path / "results.csv").exists()
    assert (tmp_path / "results.json").exists()
    assert (tmp_path / "report.md").exists()
    payload = json.loads((tmp_path / "results.json").read_text())
    assert payload["metadata"]["distances"] == [3]
    assert payload["metadata"]["physical_error_rates"] == [0.001]
    assert len(payload["results"]) == 1
```

- [ ] **Step 2: Run CLI tests to verify they fail**

Run:

```bash
python -m pytest tests/test_cli.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'decoder_benchmark.cli'`.

- [ ] **Step 3: Implement CLI**

Create `src/decoder_benchmark/cli.py` with:

```python
from __future__ import annotations

import argparse
from pathlib import Path
from typing import Sequence

from decoder_benchmark.models import LatencyStageCosts
from decoder_benchmark.reporting import ReportGenerator
from decoder_benchmark.runner import BenchmarkRunner


def parse_args(argv: Sequence[str] | None = None) -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        prog="decoder-benchmark",
        description="Run rotated surface-code MWPM benchmark sweeps with explicit latency modeling.",
    )
    parser.add_argument("--distances", nargs="+", type=int, default=[3, 5, 7])
    parser.add_argument("--error-rates", nargs="+", type=float, default=[0.001, 0.003, 0.01])
    parser.add_argument("--shots", type=int, required=True)
    parser.add_argument("--rounds", type=int, default=None)
    parser.add_argument("--basis", choices=["x", "z"], default="z")
    parser.add_argument("--output-dir", required=True)
    parser.add_argument("--seed", type=int, default=None)
    parser.add_argument("--ingestion-rounds", type=float, default=0.25)
    parser.add_argument("--graph-update-rounds", type=float, default=0.25)
    parser.add_argument("--matching-rounds", type=float, default=1.0)
    parser.add_argument("--correction-output-rounds", type=float, default=0.25)
    return parser.parse_args(argv)


def main(argv: Sequence[str] | None = None) -> int:
    args = parse_args(argv)
    latency_costs = LatencyStageCosts(
        ingestion=args.ingestion_rounds,
        graph_update=args.graph_update_rounds,
        matching=args.matching_rounds,
        correction_output=args.correction_output_rounds,
    )
    runner = BenchmarkRunner(latency_costs=latency_costs, seed=args.seed)
    results = runner.run_sweep(
        distances=args.distances,
        physical_error_rates=args.error_rates,
        shots=args.shots,
        rounds=args.rounds,
        basis=args.basis,
    )
    metadata = {
        "distances": args.distances,
        "physical_error_rates": args.error_rates,
        "shots": args.shots,
        "rounds": args.rounds if args.rounds is not None else "distance",
        "basis": args.basis,
        "seed": args.seed,
        "latency_stage_costs": latency_costs.as_dict(),
        "delegated_to_stim": "circuit generation, detector sampling, detector error models",
        "delegated_to_pymatching": "reference MWPM decoding",
        "delegated_to_sinter": "optional tiny baseline cross-check",
        "custom_code": "streaming syndrome frames, latency accounting, orchestration, reporting",
    }
    ReportGenerator(Path(args.output_dir)).write(results=results, metadata=metadata)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 4: Update README with full milestone command**

Replace `README.md` with:

```markdown
# decoder-benchmark

Simulation-first benchmark tooling for rotated planar surface-code MWPM decoding.

The milestone-one implementation delegates circuit generation and detector sampling
to Stim, reference MWPM decoding to PyMatching, and a tiny baseline cross-check to
sinter. Custom code in this repository owns the streaming-shaped syndrome frame
representation, latency accounting, benchmark orchestration, and report packaging.

## Development setup

```bash
python -m pip install -e ".[dev]"
python -m pytest -v
```

## Tiny development run

```bash
decoder-benchmark \
  --distances 3 \
  --error-rates 0.001 \
  --shots 100 \
  --rounds 3 \
  --output-dir benchmark-out/dev \
  --seed 1234
```

## Milestone sweep

```bash
decoder-benchmark \
  --distances 3 5 7 \
  --error-rates 0.001 0.003 0.01 \
  --shots 1000 \
  --output-dir benchmark-out/milestone-1
```

The command writes:

- `results.csv`: one row per distance and physical error-rate configuration
- `results.json`: structured metrics, latency breakdowns, and run metadata
- `report.md`: a readable summary table and interpretation notes

## Latency parameters

Latency is modeled in syndrome-round units and is separate from host decode wall
time. The configurable stages are:

- `--ingestion-rounds`
- `--graph-update-rounds`
- `--matching-rounds`
- `--correction-output-rounds`

## Milestone boundary

This project does not implement RTL, HLS, FPGA board integration, or a custom MWPM
engine. The latency numbers are explicit simulation parameters around a reference
software decoder; they are design aids, not hardware timing-closure evidence.
```

- [ ] **Step 5: Run CLI tests**

Run:

```bash
python -m pytest tests/test_cli.py -v
```

Expected: PASS.

- [ ] **Step 6: Run all tests**

Run:

```bash
python -m pytest -v
```

Expected: PASS.

- [ ] **Step 7: Run tiny smoke benchmark**

Run:

```bash
decoder-benchmark --distances 3 --error-rates 0.001 --shots 4 --rounds 3 --output-dir benchmark-out/smoke --seed 1234
```

Expected:

```text
benchmark-out/smoke/results.csv exists
benchmark-out/smoke/results.json exists
benchmark-out/smoke/report.md exists
```

Verify with:

```bash
test -f benchmark-out/smoke/results.csv
test -f benchmark-out/smoke/results.json
test -f benchmark-out/smoke/report.md
```

Expected: all three commands exit with status 0.

- [ ] **Step 8: Commit CLI and documentation**

Run:

```bash
git add src/decoder_benchmark/cli.py tests/test_cli.py README.md
git commit -m "feat: add benchmark cli"
```

## Task 11: Acceptance Verification

**Files:**
- Modify: no source files expected
- Verify: complete package, tests, and benchmark artifacts

- [ ] **Step 1: Run complete test suite**

Run:

```bash
python -m pytest -v
```

Expected: PASS for every test file:

```text
tests/test_circuit_factory.py
tests/test_cli.py
tests/test_decoder.py
tests/test_latency.py
tests/test_models.py
tests/test_package_import.py
tests/test_reporting.py
tests/test_runner.py
tests/test_sinter_baseline.py
tests/test_syndrome.py
```

- [ ] **Step 2: Run acceptance benchmark bundle command**

Run:

```bash
decoder-benchmark \
  --distances 3 5 7 \
  --error-rates 0.001 0.003 0.01 \
  --shots 100 \
  --output-dir benchmark-out/acceptance \
  --seed 1234
```

Expected: command exits 0 and writes `results.csv`, `results.json`, and `report.md`.

- [ ] **Step 3: Verify result row count**

Run:

```bash
python -c "import csv; rows=list(csv.DictReader(open('benchmark-out/acceptance/results.csv'))); assert len(rows) == 9, len(rows)"
```

Expected: command exits 0 because the sweep has 3 distances times 3 physical error rates.

- [ ] **Step 4: Verify JSON metadata documents dependency boundaries**

Run:

```bash
python -c "import json; data=json.load(open('benchmark-out/acceptance/results.json')); meta=data['metadata']; assert 'Stim' not in meta.get('custom_code', ''); assert 'delegated_to_stim' in meta; assert 'delegated_to_pymatching' in meta; assert 'delegated_to_sinter' in meta"
```

Expected: command exits 0.

- [ ] **Step 5: Verify Markdown report has readable summary table**

Run:

```bash
python -c "text=open('benchmark-out/acceptance/report.md').read(); assert '| distance | p_error | rounds | shots | logical_error_rate | latency_rounds |' in text; assert 'Logical error rate is measured' in text"
```

Expected: command exits 0.

- [ ] **Step 6: Commit acceptance artifacts policy only**

Do not commit generated `benchmark-out` files. If the repository later adds `.gitignore`, include `benchmark-out/` there. For this milestone, leave generated artifacts untracked unless the user explicitly asks to preserve a sample bundle.

## Self-Review

**Spec coverage:** The plan covers Stim circuit generation, detector error model extraction, PyMatching decode, streaming-shaped syndrome frames, deterministic latency in syndrome-round units, CSV/JSON/Markdown reports, configurable distance/error-rate/shot/round sweeps, and a tiny sinter baseline path. The plan explicitly excludes RTL, HLS, FPGA board work, and custom MWPM implementation.

**Placeholder scan:** The plan contains concrete paths, test code, implementation code, commands, and expected outcomes. It does not rely on unresolved implementation placeholders.

**Type consistency:** `BenchmarkConfig`, `LatencyStageCosts`, `LatencyBreakdown`, `BenchmarkResult`, `SyndromeFrame`, `MWPMDecoder`, `LatencyModel`, `ReportGenerator`, `BenchmarkRunner`, and `run_sinter_baseline` are named consistently across tests, implementation snippets, and CLI wiring.
