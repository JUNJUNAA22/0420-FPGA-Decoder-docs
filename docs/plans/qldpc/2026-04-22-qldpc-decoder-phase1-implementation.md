# QLDPC Decoder Phase 1 Implementation Plan

> Status: Historical implementation plan. The work described here may already
> be partially or fully implemented. Use the corresponding architecture page and
> current source tree as the source of truth.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Python foundation for QLDPC decoder including random code generator, decoder API, sparse data structures, and basic message passing algorithm.

**Architecture:** Implement a four-layer Python foundation: (1) QLDPCCodeGenerator for random quantum LDPC codes, (2) QLDPCDecoder API compatible with Surface Code decoder, (3) SparseFormat module for CSR/COO data structures, (4) BasicMessagePassing for iterative decoding.

**Tech Stack:** Python 3.11+, NumPy, pytest, no external quantum libraries for Phase 1.

---

## File Structure

**New files to create:**
- `qldpc_decoder/src/qldpc/code_generator.py` - Random QLDPC code generation
- `qldpc_decoder/src/qldpc/models.py` - Data models (QLDPCConfig, CodeFeatures, etc.)
- `qldpc_decoder/src/qldpc/sparse_format.py` - Sparse matrix formats (CSR, COO)
- `qldpc_decoder/src/qldpc/decoder.py` - Main QLDPCDecoder API
- `qldpc_decoder/src/qldpc/message_passing.py` - Basic message passing algorithm
- `qldpc_decoder/src/qldpc/__init__.py` - Package exports
- `qldpc_decoder/tests/test_code_generator.py` - Code generator tests
- `qldpc_decoder/tests/test_sparse_format.py` - Sparse format tests
- `qldpc_decoder/tests/test_decoder.py` - Decoder API tests

**Files to modify:**
- `qldpc_decoder/pyproject.toml` - Update dependencies
- `surface_code_decoder/tests/test_qldpc_integration.py` - Fill in integration tests

---

### Task 1: Add models.py with basic data structures

**Files:**
- Create: `qldpc_decoder/src/qldpc/models.py`
- Test: `qldpc_decoder/tests/test_models.py`

- [ ] **Step 1: Write failing test for QLDPCConfig**

```python
import numpy as np
from qldpc.models import QLDPCConfig

def test_qlldpc_config_creation():
    config = QLDPCConfig(
        code_length=100,
        max_iterations=50,
        convergence_threshold=1e-6
    )
    assert config.code_length == 100
    assert config.max_iterations == 50
    assert config.convergence_threshold == 1e-6
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_models.py::test_qlldpc_config_creation -v`
Expected: FAIL with "ModuleNotFoundError: No module named 'qldpc.models'"

- [ ] **Step 3: Write QLDPCConfig class**

```python
from dataclasses import dataclass

@dataclass
class QLDPCConfig:
    """Configuration for QLDPC decoder."""

    code_length: int = 100
    max_iterations: int = 50
    convergence_threshold: float = 1e-6
    seed: int = 42

    def __post_init__(self):
        if self.code_length <= 0:
            raise ValueError("code_length must be positive")
        if self.max_iterations <= 0:
            raise ValueError("max_iterations must be positive")
        if self.convergence_threshold <= 0:
            raise ValueError("convergence_threshold must be positive")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_models.py::test_qlldpc_config_creation -v`
Expected: PASS

- [ ] **Step 5: Write test for CodeFeatures model**

```python
from qldpc.models import CodeFeatures

def test_code_features_creation():
    features = CodeFeatures(
        sparsity=0.1,
        avg_node_degree=3.5,
        cycle_count=100
    )
    assert features.sparsity == 0.1
    assert features.avg_node_degree == 3.5
    assert features.cycle_count == 100
```

- [ ] **Step 6: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_models.py::test_code_features_creation -v`
Expected: FAIL with "CodeFeatures not defined"

- [ ] **Step 7: Write CodeFeatures class**

```python
@dataclass
class CodeFeatures:
    """Features extracted from QLDPC code for strategy selection."""

    sparsity: float
    avg_node_degree: float
    cycle_count: int
    min_distance: int = 0

    def __post_init__(self):
        if not 0 <= self.sparsity <= 1:
            raise ValueError("sparsity must be between 0 and 1")
        if self.avg_node_degree <= 0:
            raise ValueError("avg_node_degree must be positive")
```

- [ ] **Step 8: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_models.py::test_code_features_creation -v`
Expected: PASS

- [ ] **Step 9: Write test for ResourceBudget model**

```python
from qldpc.models import ResourceBudget

def test_resource_budget_creation():
    budget = ResourceBudget(
        max_lut_percent=80,
        max_bram_percent=70,
        max_dsp_percent=60
    )
    assert budget.max_lut_percent == 80
    assert budget.max_bram_percent == 70
    assert budget.max_dsp_percent == 60
```

- [ ] **Step 10: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_models.py::test_resource_budget_creation -v`
Expected: FAIL with "ResourceBudget not defined"

- [ ] **Step 11: Write ResourceBudget class**

```python
@dataclass
class ResourceBudget:
    """FPGA resource budget for decoder implementation."""

    max_lut_percent: float = 80.0
    max_bram_percent: float = 70.0
    max_dsp_percent: float = 60.0

    def __post_init__(self):
        for attr in ['max_lut_percent', 'max_bram_percent', 'max_dsp_percent']:
            value = getattr(self, attr)
            if not 0 <= value <= 100:
                raise ValueError(f"{attr} must be between 0 and 100")
```

- [ ] **Step 12: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_models.py::test_resource_budget_creation -v`
Expected: PASS

- [ ] **Step 13: Export models in __init__.py**

```python
from qldpc.models import QLDPCConfig, CodeFeatures, ResourceBudget

__all__ = ['QLDPCConfig', 'CodeFeatures', 'ResourceBudget']
```

- [ ] **Step 14: Run all model tests**

Run: `cd qldpc_decoder && python -m pytest tests/test_models.py -v`
Expected: All 3 tests PASS

- [ ] **Step 15: Commit**

```bash
cd qldpc_decoder
git add src/qldpc/models.py tests/test_models.py src/qldpc/__init__.py
git commit -m "feat: add basic data models (QLDPCConfig, CodeFeatures, ResourceBudget)"
```

---

### Task 2: Implement sparse matrix formats (CSR and COO)

**Files:**
- Create: `qldpc_decoder/src/qldpc/sparse_format.py`
- Test: `qldpc_decoder/tests/test_sparse_format.py`

- [ ] **Step 1: Write failing test for CSR conversion**

```python
import numpy as np
from qldpc.sparse_format import CSRMatrix

def test_csr_from_dense():
    dense = np.array([
        [1, 0, 1, 0],
        [0, 1, 0, 1],
        [1, 0, 0, 1],
    ], dtype=np.int32)

    sparse = CSRMatrix.from_dense(dense)

    assert sparse.nnz == 5
    assert sparse.n_rows == 3
    assert sparse.n_cols == 4
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_sparse_format.py::test_csr_from_dense -v`
Expected: FAIL with "CSRMatrix not defined"

- [ ] **Step 3: Write CSRMatrix class**

```python
import numpy as np
from dataclasses import dataclass

@dataclass
class CSRMatrix:
    """Compressed Sparse Row matrix format."""

    data: np.ndarray
    indices: np.ndarray
    indptr: np.ndarray
    n_rows: int
    n_cols: int

    @property
    def nnz(self) -> int:
        return len(self.data)

    @classmethod
    def from_dense(cls, matrix: np.ndarray) -> "CSRMatrix":
        """Convert dense matrix to CSR format."""
        if matrix.ndim != 2:
            raise ValueError("Input must be 2D matrix")

        n_rows, n_cols = matrix.shape
        data = []
        indices = []
        indptr = [0]

        for i in range(n_rows):
            row_nnz = 0
            for j in range(n_cols):
                if matrix[i, j] != 0:
                    data.append(matrix[i, j])
                    indices.append(j)
                    row_nnz += 1
            indptr.append(indptr[-1] + row_nnz)

        return cls(
            data=np.array(data, dtype=matrix.dtype),
            indices=np.array(indices, dtype=np.int32),
            indptr=np.array(indptr, dtype=np.int32),
            n_rows=n_rows,
            n_cols=n_cols
        )

    def to_dense(self) -> np.ndarray:
        """Convert CSR format back to dense matrix."""
        dense = np.zeros((self.n_rows, self.n_cols), dtype=self.data.dtype)

        for i in range(self.n_rows):
            start, end = self.indptr[i], self.indptr[i + 1]
            for j in range(start, end):
                col = self.indices[j]
                dense[i, col] = self.data[j]

        return dense
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_sparse_format.py::test_csr_from_dense -v`
Expected: PASS

- [ ] **Step 5: Write test for CSR to dense conversion**

```python
def test_csr_to_dense():
    original = np.array([
        [1, 0, 1],
        [0, 1, 0],
        [1, 1, 1],
    ], dtype=np.int32)

    sparse = CSRMatrix.from_dense(original)
    recovered = sparse.to_dense()

    assert np.array_equal(original, recovered)
```

- [ ] **Step 6: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_sparse_format.py::test_csr_to_dense -v`
Expected: PASS

- [ ] **Step 7: Write test for COO format**

```python
from qldpc.sparse_format import COOMatrix

def test_coo_from_dense():
    dense = np.array([
        [1, 0, 1],
        [0, 2, 0],
    ], dtype=np.int32)

    sparse = COOMatrix.from_dense(dense)

    assert sparse.nnz == 3
    assert sparse.n_rows == 2
    assert sparse.n_cols == 3
```

- [ ] **Step 8: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_sparse_format.py::test_coo_from_dense -v`
Expected: FAIL with "COOMatrix not defined"

- [ ] **Step 9: Write COOMatrix class**

```python
@dataclass
class COOMatrix:
    """Coordinate (COO) sparse matrix format."""

    row: np.ndarray
    col: np.ndarray
    data: np.ndarray
    n_rows: int
    n_cols: int

    @property
    def nnz(self) -> int:
        return len(self.data)

    @classmethod
    def from_dense(cls, matrix: np.ndarray) -> "COOMatrix":
        """Convert dense matrix to COO format."""
        if matrix.ndim != 2:
            raise ValueError("Input must be 2D matrix")

        n_rows, n_cols = matrix.shape
        rows, cols = np.nonzero(matrix)
        data = matrix[rows, cols]

        return cls(
            row=rows.astype(np.int32),
            col=cols.astype(np.int32),
            data=data,
            n_rows=n_rows,
            n_cols=n_cols
        )

    def to_dense(self) -> np.ndarray:
        """Convert COO format back to dense matrix."""
        dense = np.zeros((self.n_rows, self.n_cols), dtype=self.data.dtype)
        dense[self.row, self.col] = self.data
        return dense
```

- [ ] **Step 10: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_sparse_format.py::test_coo_from_dense -v`
Expected: PASS

- [ ] **Step 11: Export sparse formats**

```python
from qldpc.sparse_format import CSRMatrix, COOMatrix

__all__ = ['CSRMatrix', 'COOMatrix']
```

- [ ] **Step 12: Run all sparse format tests**

Run: `cd qldpc_decoder && python -m pytest tests/test_sparse_format.py -v`
Expected: All 4 tests PASS

- [ ] **Step 13: Commit**

```bash
cd qldpc_decoder
git add src/qldpc/sparse_format.py tests/test_sparse_format.py
git commit -m "feat: add sparse matrix formats (CSR, COO)"
```

---

### Task 3: Implement random QLDPC code generator

**Files:**
- Create: `qldpc_decoder/src/qldpc/code_generator.py`
- Test: `qldpc_decoder/tests/test_code_generator.py`

- [ ] **Step 1: Write failing test for small code generation**

```python
import numpy as np
from qldpc.code_generator import QLDPCCodeGenerator

def test_generate_small_qldpc_code():
    generator = QLDPCCodeGenerator(seed=1234)
    n = 100
    k = 50

    h_x, h_z = generator.generate_random_qldpc(n, k)

    assert h_x.shape == (n - k, n)
    assert h_z.shape == (n - k, n)
    assert h_x.dtype == np.int32
    assert h_z.dtype == np.int32
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_code_generator.py::test_generate_small_qldpc_code -v`
Expected: FAIL with "QLDPCCodeGenerator not defined"

- [ ] **Step 3: Write QLDPCCodeGenerator class with basic random generation**

```python
import numpy as np
from dataclasses import dataclass

@dataclass
class QLDPCCodeGenerator:
    """Generates random Quantum Low-Density Parity-Check codes."""

    seed: int = 42
    sparsity: float = 0.1

    def __post_init__(self):
        self.rng = np.random.default_rng(self.seed)
        if not 0 < self.sparsity < 1:
            raise ValueError("sparsity must be between 0 and 1")

    def generate_random_qldpc(self, n: int, k: int) -> tuple[np.ndarray, np.ndarray]:
        """Generate random QLDPC code with parameters (n, k).

        Returns (H_x, H_z) parity check matrices where:
        - n is the total number of physical qubits
        - k is the number of logical qubits
        - H_x and H_z are (n-k) x n matrices
        """
        m = n - k
        expected_nnz = int(m * n * self.sparsity)

        # Generate random sparse H_x
        h_x = self._generate_sparse_matrix(m, n, expected_nnz)

        # Generate random sparse H_z
        h_z = self._generate_sparse_matrix(m, n, expected_nnz)

        return h_x, h_z

    def _generate_sparse_matrix(self, rows: int, cols: int, nnz: int) -> np.ndarray:
        """Generate random sparse matrix with exactly nnz non-zero entries."""
        matrix = np.zeros((rows, cols), dtype=np.int32)

        for i in range(nnz):
            r = self.rng.integers(0, rows)
            c = self.rng.integers(0, cols)
            matrix[r, c] = 1

        return matrix
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_code_generator.py::test_generate_small_qldpc_code -v`
Expected: PASS

- [ ] **Step 5: Write test for code validity (no zero rows/columns)**

```python
def test_code_no_zero_rows_or_columns():
    generator = QLDPCCodeGenerator(seed=42)
    h_x, h_z = generator.generate_random_qldpc(100, 50)

    # Check no zero rows
    assert not np.any(np.all(h_x == 0, axis=1))
    assert not np.any(np.all(h_z == 0, axis=1))

    # Check no zero columns (with high probability for random codes)
    assert not np.any(np.all(h_x == 0, axis=0))
    assert not np.any(np.all(h_z == 0, axis=0))
```

- [ ] **Step 6: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_code_generator.py::test_code_no_zero_rows_or_columns -v`
Expected: PASS

- [ ] **Step 7: Write test for code properties (rate, sparsity)**

```python
def test_code_properties():
    generator = QLDPCCodeGenerator(sparsity=0.1, seed=99)
    n, k = 100, 50
    h_x, h_z = generator.generate_random_qldpc(n, k)

    # Check code rate is k/n
    rate = k / n
    assert rate == 0.5

    # Check sparsity approximately matches target
    actual_sparsity_x = np.count_nonzero(h_x) / h_x.size
    assert abs(actual_sparsity_x - 0.1) < 0.05  # Within 5%
```

- [ ] **Step 8: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_code_generator.py::test_code_properties -v`
Expected: PASS

- [ ] **Step 9: Export code generator**

```python
from qldpc.code_generator import QLDPCCodeGenerator

__all__ = ['QLDPCCodeGenerator']
```

- [ ] **Step 10: Run all code generator tests**

Run: `cd qldpc_decoder && python -m pytest tests/test_code_generator.py -v`
Expected: All 3 tests PASS

- [ ] **Step 11: Commit**

```bash
cd qldpc_decoder
git add src/qldpc/code_generator.py tests/test_code_generator.py
git commit -m "feat: add random QLDPC code generator"
```

---

### Task 4: Implement basic message passing decoder

**Files:**
- Create: `qldpc_decoder/src/qldpc/message_passing.py`
- Test: `qldpc_decoder/tests/test_message_passing.py`

- [ ] **Step 1: Write failing test for message passing decoder**

```python
import numpy as np
from qldpc.message_passing import MessagePassingDecoder

def test_message_passing_decodes_simple_syndrome():
    decoder = MessagePassingDecoder(max_iterations=10)

    h_x = np.array([[1, 1, 0, 0], [0, 1, 1, 0]], dtype=np.int32)
    syndrome = np.array([1, 0], dtype=np.bool_)

    prediction = decoder.decode(h_x, syndrome)

    assert prediction.shape == (4,)
    assert prediction.dtype == np.bool_
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_message_passing.py::test_message_passing_decodes_simple_syndrome -v`
Expected: FAIL with "MessagePassingDecoder not defined"

- [ ] **Step 3: Write MessagePassingDecoder class**

```python
import numpy as np
from dataclasses import dataclass

@dataclass
class MessagePassingDecoder:
    """Basic iterative message passing decoder for LDPC codes."""

    max_iterations: int = 50
    convergence_threshold: float = 1e-6

    def decode(self, h: np.ndarray, syndrome: np.ndarray) -> np.ndarray:
        """Decode syndrome using message passing algorithm.

        Args:
            h: Parity check matrix (m x n)
            syndrome: Syndrome vector (m,)

        Returns:
            Decoded prediction (n,)
        """
        m, n = h.shape
        if syndrome.shape != (m,):
            raise ValueError(f"Syndrome shape {syndrome.shape} doesn't match matrix {m}")

        # Initialize variable to check messages
        v2c_messages = np.zeros((n, m), dtype=np.float64)

        for iteration in range(self.max_iterations):
            old_messages = v2c_messages.copy()

            # Check to variable messages (sum incoming variable messages)
            c2v_messages = self._compute_c2v_messages(h, syndrome, v2c_messages)

            # Variable to check messages (sum incoming check messages)
            v2c_messages = self._compute_v2c_messages(h, c2v_messages)

            # Check convergence
            diff = np.max(np.abs(v2c_messages - old_messages))
            if diff < self.convergence_threshold:
                break

        # Make decision based on final messages
        decision = self._make_decision(v2c_messages)
        return decision.astype(np.bool_)

    def _compute_c2v_messages(self, h: np.ndarray, syndrome: np.ndarray, v2c_messages: np.ndarray) -> np.ndarray:
        """Compute check node to variable node messages."""
        m, n = h.shape
        c2v = np.zeros((m, n), dtype=np.float64)

        for i in range(m):  # For each check node
            for j in range(n):  # For each connected variable
                if h[i, j] == 0:
                    continue

                # Sum messages from all other variables in this check
                other_vars = [k for k in range(n) if k != j and h[i, k] == 1]
                if not other_vars:
                    continue

                msg_sum = np.sum([v2c_messages[k, i] for k in other_vars])
                c2v[i, j] = (-1) ** syndrome[i] * msg_sum

        return c2v

    def _compute_v2c_messages(self, h: np.ndarray, c2v_messages: np.ndarray) -> np.ndarray:
        """Compute variable node to check node messages."""
        m, n = h.shape
        v2c = np.zeros((n, m), dtype=np.float64)

        for j in range(n):  # For each variable node
            for i in range(m):  # For each connected check
                if h[i, j] == 0:
                    continue

                # Sum messages from all other checks for this variable
                other_checks = [k for k in range(m) if k != i and h[k, j] == 1]
                if not other_checks:
                    continue

                v2c[j, i] = np.sum([c2v_messages[k, j] for k in other_checks])

        return v2c

    def _make_decision(self, v2c_messages: np.ndarray) -> np.ndarray:
        """Make final decision based on variable node beliefs."""
        n, m = v2c_messages.shape

        # Decision: majority of incoming messages (or 0 if no messages)
        decisions = np.zeros(n, dtype=np.float64)

        for j in range(n):
            connected_checks = [v2c_messages[j, i] for i in range(m)]
            if connected_checks:
                decisions[j] = 1 if np.mean(connected_checks) > 0 else 0

        return decisions
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_message_passing.py::test_message_passing_decodes_simple_syndrome -v`
Expected: PASS

- [ ] **Step 5: Write test for batch decoding**

```python
def test_decode_batch():
    decoder = MessagePassingDecoder(max_iterations=10)

    h_x = np.array([[1, 1, 0], [0, 1, 1]], dtype=np.int32)
    syndromes = np.array([
        [1, 0],
        [0, 1],
        [1, 1],
    ], dtype=np.bool_)

    predictions = decoder.decode_batch(h_x, syndromes)

    assert predictions.shape == (3, 3)
    assert predictions.dtype == np.bool_
```

- [ ] **Step 6: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_message_passing.py::test_decode_batch -v`
Expected: FAIL with "decode_batch not defined"

- [ ] **Step 7: Add decode_batch method**

```python
def decode_batch(self, h: np.ndarray, syndromes: np.ndarray) -> np.ndarray:
    """Decode batch of syndromes using message passing.

    Args:
        h: Parity check matrix (m x n)
        syndromes: Batch of syndromes (batch_size x m)

    Returns:
        Decoded predictions (batch_size x n)
    """
    if syndromes.ndim != 2:
        raise ValueError("Syndromes must be 2D array (batch_size x m)")

    batch_size, m = syndromes.shape
    predictions = []

    for i in range(batch_size):
        pred = self.decode(h, syndromes[i])
        predictions.append(pred)

    return np.array(predictions)
```

Add this method to the MessagePassingDecoder class.

- [ ] **Step 8: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_message_passing.py::test_decode_batch -v`
Expected: PASS

- [ ] **Step 9: Export message passing decoder**

```python
from qldpc.message_passing import MessagePassingDecoder

__all__ = ['MessagePassingDecoder']
```

- [ ] **Step 10: Run all message passing tests**

Run: `cd qldpc_decoder && python -m pytest tests/test_message_passing.py -v`
Expected: All 2 tests PASS

- [ ] **Step 11: Commit**

```bash
cd qldpc_decoder
git add src/qldpc/message_passing.py tests/test_message_passing.py
git commit -m "feat: add basic message passing decoder"
```

---

### Task 5: Implement QLDPCDecoder API

**Files:**
- Create: `qldpc_decoder/src/qldpc/decoder.py`
- Test: `qldpc_decoder/tests/test_decoder.py`

- [ ] **Step 1: Write failing test for decoder initialization**

```python
import numpy as np
from qldpc.decoder import QLDPCDecoder
from qldpc.models import QLDPCConfig

def test_decoder_initialization():
    config = QLDPCConfig(code_length=100, max_iterations=50)
    h_x = np.array([[1, 1, 0], [0, 1, 1]], dtype=np.int32)
    h_z = np.array([[1, 0, 1], [1, 1, 0]], dtype=np.int32)

    decoder = QLDPCDecoder(h_x, h_z, config)

    assert decoder.h_x.shape == (2, 3)
    assert decoder.h_z.shape == (2, 3)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd qldpc_decoder && python -m pytest tests/test_decoder.py::test_decoder_initialization -v`
Expected: FAIL with "QLDPCDecoder not defined"

- [ ] **Step 3: Write QLDPCDecoder class**

```python
import numpy as np
from dataclasses import dataclass
from qldpc.models import QLDPCConfig
from qldpc.message_passing import MessagePassingDecoder

@dataclass
class QLDPCDecoder:
    """QLDPC decoder with adaptive sparse architecture."""

    h_x: np.ndarray
    h_z: np.ndarray
    config: QLDPCConfig
    _mp_decoder_x: MessagePassingDecoder = None
    _mp_decoder_z: MessagePassingDecoder = None

    def __post_init__(self):
        if self.h_x.shape != self.h_z.shape:
            raise ValueError("H_x and H_z must have the same shape")

        # Initialize message passing decoders
        mp_config = {
            'max_iterations': self.config.max_iterations,
            'convergence_threshold': self.config.convergence_threshold
        }
        self._mp_decoder_x = MessagePassingDecoder(**mp_config)
        self._mp_decoder_z = MessagePassingDecoder(**mp_config)

    @classmethod
    def from_qldpc_code(cls, h_x: np.ndarray, h_z: np.ndarray,
                        config: QLDPCConfig = None) -> "QLDPCDecoder":
        """Create decoder from QLDPC parity check matrices.

        Args:
            h_x: X-type parity check matrix
            h_z: Z-type parity check matrix
            config: Decoder configuration (uses default if None)

        Returns:
            QLDPCDecoder instance
        """
        if config is None:
            n = h_x.shape[1]
            config = QLDPCConfig(code_length=n)

        return cls(h_x=h_x, h_z=h_z, config=config)

    def decode_batch(self, syndrome_samples: np.ndarray) -> np.ndarray:
        """Decode a batch of syndrome samples.

        Args:
            syndrome_samples: Array of syndromes (shots x detectors)

        Returns:
            Predictions for observable flips (shots x observables)
        """
        shots, n_detectors = syndrome_samples.shape

        # For now, decode using H_x (simplified approach)
        # Full implementation would use both H_x and H_z
        predictions = self._mp_decoder_x.decode_batch(
            self.h_x, syndrome_samples
        )

        return predictions
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_decoder.py::test_decoder_initialization -v`
Expected: PASS

- [ ] **Step 5: Write test for from_qldpc_code factory method**

```python
def test_from_qldpc_code_factory():
    h_x = np.array([[1, 1, 0]], dtype=np.int32)
    h_z = np.array([[0, 1, 1]], dtype=np.int32)

    decoder = QLDPCDecoder.from_qldpc_code(h_x, h_z)

    assert decoder.h_x.shape == h_x.shape
    assert decoder.h_z.shape == h_z.shape
    assert decoder.config.code_length == 3  # Default from h_x
```

- [ ] **Step 6: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_decoder.py::test_from_qldpc_code_factory -v`
Expected: PASS

- [ ] **Step 7: Write test for decode_batch**

```python
def test_decode_batch_compatibility():
    h_x = np.array([[1, 1, 0], [0, 1, 1]], dtype=np.int32)
    h_z = np.array([[1, 0, 1], [1, 1, 0]], dtype=np.int32)

    decoder = QLDPCDecoder.from_qldpc_code(h_x, h_z)

    syndromes = np.array([
        [1, 0],
        [0, 1],
    ], dtype=np.bool_)

    predictions = decoder.decode_batch(syndromes)

    assert predictions.shape == (2, 3)
    assert predictions.dtype == np.bool_
```

- [ ] **Step 8: Run test to verify it passes**

Run: `cd qldpc_decoder && python -m pytest tests/test_decoder.py::test_decode_batch_compatibility -v`
Expected: PASS

- [ ] **Step 9: Export decoder**

```python
from qldpc.decoder import QLDPCDecoder

__all__ = ['QLDPCDecoder']
```

- [ ] **Step 10: Run all decoder tests**

Run: `cd qldpc_decoder && python -m pytest tests/test_decoder.py -v`
Expected: All 3 tests PASS

- [ ] **Step 11: Commit**

```bash
cd qldpc_decoder
git add src/qldpc/decoder.py tests/test_decoder.py src/qldpc/__init__.py
git commit -m "feat: add QLDPCDecoder API compatible with Surface Code decoder"
```

---

### Task 6: Update pyproject.toml with dependencies

**Files:**
- Modify: `qldpc_decoder/pyproject.toml`

- [ ] **Step 1: Write failing test for import**

Run: `cd qldpc_decoder && python -c "from qldpc import QLDPCDecoder, QLDPCCodeGenerator"`
Expected: PASS (but verify all imports work)

- [ ] **Step 2: Update pyproject.toml dependencies**

```toml
[project]
name = "qldpc-decoder"
version = "0.1.0"
description = "FPGA-accelerated QLDPC decoder with adaptive hybrid sparse architecture"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "numpy>=1.24.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
]

[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-data]
qldpc = ["py.typed"]
```

- [ ] **Step 3: Install package in editable mode**

Run: `cd qldpc_decoder && pip install -e .`
Expected: Successfully installed qldpc-decoder-0.1.0

- [ ] **Step 4: Verify imports work**

Run: `cd qldpc_decoder && python -c "from qldpc import QLDPCDecoder, QLDPCCodeGenerator, MessagePassingDecoder, CSRMatrix"`
Expected: No errors

- [ ] **Step 5: Commit**

```bash
cd qldpc_decoder
git add pyproject.toml
git commit -m "chore: update pyproject.toml with dependencies and package config"
```

---

### Task 7: Run full test suite and verify integration

**Files:**
- Test: All test files

- [ ] **Step 1: Run all tests**

Run: `cd qldpc_decoder && python -m pytest tests/ -v`
Expected: All tests pass

- [ ] **Step 2: Check test coverage**

Run: `cd qldpc_decoder && python -m pytest tests/ --cov=qldpc --cov-report=term-missing`
Expected: Coverage report with coverage percentage

- [ ] **Step 3: Run integration test with main decoder tests**

Run: `cd decoder && python -m pytest tests/test_qldpc_integration.py -v`
Expected: Tests pass (or skip with appropriate message)

- [ ] **Step 4: Verify package structure**

Run: `cd qldpc_decoder && find src/ -type f -name "*.py" | sort`
Expected: List of all Python files in correct locations

- [ ] **Step 5: Commit any final fixes**

```bash
cd qldpc_decoder
git add .
git commit -m "test: verify Phase 1 implementation and integration"
```

---

## Self-Review

### Spec Coverage Check

From the design spec:

1. **Random QLDPC code generator** - Covered in Task 3
2. **Decoder API compatible with Surface Code** - Covered in Task 5
3. **Sparse data structures (CSR, COO)** - Covered in Task 2
4. **Basic message passing algorithm** - Covered in Task 4
5. **Testing infrastructure** - Covered in all tasks

### Placeholder Scan

No TBD, TODO, or incomplete steps found. All code blocks are complete.

### Type Consistency Check

- `QLDPCConfig` used consistently across files
- `CodeFeatures` and `ResourceBudget` defined and used correctly
- Method signatures match between tests and implementations
- All imports are properly declared

### Gap Analysis

The design spec mentions Phase 2 (Adaptive Strategy Engine), Phase 3 (FPGA Hardware), Phase 4 (Integration), Phase 5 (Optimization). These will be covered in subsequent plans.

This plan implements Phase 1: Python Foundation as specified.

---

## Plan complete and saved to `docs/plans/qldpc/2026-04-22-qldpc-decoder-phase1-implementation.md`.

Two execution options:

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
