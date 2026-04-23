# QLDPC Adaptive Hybrid Sparse Decoder Design

## Goal

Design and implement an FPGA-based quantum Low-Density Parity-Check (QLDPC) decoder with adaptive hybrid sparse architecture that achieves real-time decoding performance (millisecond-level latency) for code lengths up to n=1000.

## Scope

This design covers:

- Implementing a random QLDPC code generator
- Designing an adaptive hybrid sparse architecture for parallel decoding
- Creating FPGA hardware for sparse message passing and parallel computation
- Defining Python API compatible with existing Surface Code decoder
- Implementing data flow between Python host and FPGA accelerator
- Verification and testing infrastructure

This design does not cover:

- FPGA synthesis and bitstream generation for specific board
- Physical quantum hardware integration
- Quantum error correction protocol implementation beyond decoding

## Why Adaptive Hybrid Sparse Architecture

Existing QLDPC decoder approaches face trade-offs between parallelism, resource utilization, and decoding accuracy:

- **Pure Belief Propagation**: High parallelism but may not converge, requires many iterations
- **BP + OSD**: Better accuracy but higher latency due to OSD phase
- **Fully parallel hardware**: Maximum throughput but resource-prohibitive for n=1000

The adaptive hybrid sparse architecture addresses these by:
- Dynamically selecting sparse strategies based on code characteristics
- Balancing parallelism with FPGA resource usage
- Achieving millisecond-level latency while maintaining high accuracy
- Reference: Vegapunk's hierarchical and sparse acceleration concepts

## Architecture

The system has four layers.

### 1. Python API Layer

Provides decoder interface compatible with existing Surface Code implementation:

```python
class QLDPCDecoder:
    """QLDPC decoder with FPGA acceleration"""

    def __init__(self, config: QLDPCConfig) -> None:
        """Initialize decoder with sparse architecture config"""

    def from_qldpc_code(cls, h_x: np.ndarray, h_z: np.ndarray) -> "QLDPCDecoder":
        """Create decoder from QLDPC parity check matrices"""

    def decode_batch(self, syndrome_samples: np.ndarray) -> np.ndarray:
        """Decode syndrome batch using FPGA acceleration"""
```

### 2. Adaptive Sparse Strategy Engine

Analyzes code features and selects optimal sparse strategies:

```python
class AdaptiveSparseStrategy:
    """Engine for adaptive sparse strategy selection"""

    def analyze_code(self, h: np.ndarray) -> CodeFeatures:
        """Analyze QLDPC code characteristics:
        - Sparsity pattern
        - Node degree distribution
        - Cycle structure
        - Resource estimation
        """

    def select_strategy(self, features: CodeFeatures, resources: ResourceBudget) -> SparseStrategy:
        """Select optimal sparse strategy:
        - Graph pruning level
        - Zero-value filtering threshold
        - Data compression format
        - Parallelization degree
        """

    def adapt_during_decoding(self, progress: DecodingProgress) -> StrategyUpdate:
        """Adapt strategy during decoding iterations"""
```

### 3. FPGA Control Layer

Manages data flow between host and FPGA:

```verilog
module fpga_controller (
    input clk,
    input rst_n,
    // Host interface
    input  [31:0] host_cmd,
    output [31:0] host_status,
    // Memory interface
    output [31:0] ddr_addr,
    output [31:0] ddr_wdata,
    input  [31:0] ddr_rdata,
    output        ddr_we,
    // Compute core interface
    output        core_start,
    input         core_done
);
```

Responsibilities:
- Block scheduling for large codes (n=1000)
- Streaming data buffering
- Sparse data structure management
- Resource monitoring

### 4. FPGA Compute Layer

Parallel sparse decoding cores:

```verilog
module sparse_decoder_core (
    input  clk,
    input  rst_n,
    // Sparse Tanner graph interface
    input  [15:0] edge_count,
    input  [31:0] edge_mem_base,
    // Message passing interface
    input  msg_valid,
    output msg_ready,
    input  [31:0] msg_data,
    // Result interface
    output [15:0] prediction,
    output        valid
);
```

Components:
- Sparse Message Passing Unit (SMPU)
- Parallel Check Node Processor (PCNP)
- Sparse Matrix Storage Engine (SMSE)

## Data Flow

The end-to-end flow is:

```
Python QLDPC code generation
  -> Random QLDPC parity check matrices (Hx, Hz)
  -> Syndrome sampling
  -> Adaptive strategy engine analyzes code
  -> Optimal sparse strategy selected
  -> Sparse data structures prepared (CSR format)
  -> FPGA control layer schedules blocks
  -> Streaming data to FPGA via PCIe/Ethernet
  -> FPGA compute layer executes parallel sparse decoding
  -> Results aggregated and streamed back
  -> Predictions returned to Python API
```

## Sparse Strategies

### 1. Graph Structure Sparsification

- **Preprocessing**: Static analysis and low-weight edge pruning
- **Runtime**: Dynamic edge re-weighting based on convergence
- **Format**: CSR (Compressed Sparse Row) for Tanner graph storage

### 2. Computation Sparsification

- **Zero-value detection**: Hardware-level zero detection and skipping
- **Threshold filtering**: Skip messages below adaptive threshold
- **Early termination**: Stop iterations if convergence detected

### 3. Data Sparsification

- **Adaptive compression**: Dynamic sparse data structure
- **Delta encoding**: For sequential syndrome patterns
- **Run-length encoding**: For repeated values

### 4. Adaptive Strategy Selection

The strategy engine selects combination based on:
- Code complexity (sparsity pattern, cycle structure)
- Resource availability (LUT, BRAM, DSP usage)
- Latency requirement (strict vs relaxed)
- Accuracy target (error rate threshold)

## FPGA Resource Estimation

Based on Xilinx Alveo U250 (or similar):

| Component   | LUT      | BRAM     | DSP      | Notes           |
|-------------|----------|----------|----------|-----------------|
| SMPU        | 15-20%   | 10-15%   | 10-15%   | Message passing |
| PCNP        | 20-25%   | 15-20%   | 15-20%   | Check nodes     |
| SMSE        | 10-15%   | 20-25%   | 5-10%    | Sparse storage  |
| Controller  | 10-15%   | 5-10%    | 0%       | Data flow       |
| Strategy    | 10-15%   | 5-10%    | 5-10%    | Adaptation      |
| **Total**   | **65-90%**| **55-80%**| **35-55%**|                 |

**Target Performance:**
- Latency: 0.8-1.5 ms (millisecond-level for neutral atom experiments)
- Throughput: 1000+ shots/second
- Accuracy: Comparable to BP-based decoders

## Directory Structure

```
0420-FPGA-surface_code_decoder/
├── qldpc_decoder/               # New QLDPC decoder module
│   ├── src/
│   │   ├── qldpc/
│   │   │   ├── code_generator.py    # Random QLDPC code generator
│   │   │   ├── decoder.py            # Main QLDPC decoder API
│   │   │   ├── adaptive_strategy.py # Adaptive sparse strategy engine
│   │   │   ├── sparse_format.py     # Sparse data structures
│   │   │   └── fpga_interface.py    # FPGA host interface
│   │   └── fpga/
│   │       ├── fpga_controller.v    # FPGA control layer
│   │       ├── sparse_core.v        # Sparse decoder core
│   │       ├── smpu.v               # Sparse message passing unit
│   │       ├── pcnp.v               # Parallel check node processor
│   │       └── smse.v               # Sparse matrix storage engine
│   ├── tests/
│   │   ├── test_code_generator.py  # Test code generation
│   │   ├── test_adaptive_strategy.py # Test strategy selection
│   │   ├── test_sparse_format.py    # Test sparse formats
│   │   └── test_integration.py      # Integration tests
│   ├── docs/
│   │   └── design_notes/            # Design documentation
│   └── pyproject.toml               # Python package config
├── verilog_test/
│   └── qldpc_test/                  # FPGA testbench for QLDPC
└── docs/specs/qldpc/2026-04-22-qldpc-adaptive-sparse-decoder-design.md
```

## Implementation Phases

### Phase 1: Python Foundation
- Implement random QLDPC code generator
- Create decoder API compatible with Surface Code
- Implement sparse data structures (CSR, COO)
- Implement basic message passing algorithm

### Phase 2: Adaptive Strategy Engine
- Implement code feature analysis
- Implement strategy selection logic
- Add resource monitoring hooks
- Implement strategy adaptation during decoding

### Phase 3: FPGA Hardware Design
- Design FPGA controller module
- Implement sparse decoder core
- Implement SMPU, PCNP, SMSE modules
- Create testbench for verification

### Phase 4: Integration
- Integrate Python host interface with FPGA
- Implement data streaming
- Add performance monitoring
- Create integration tests

### Phase 5: Optimization
- Profile and optimize sparse strategies
- Tune adaptation parameters
- Optimize FPGA resource usage
- Performance benchmarking

## Testing Strategy

### Unit Tests
- Code generator correctness
- Sparse format validity
- Strategy engine logic
- Individual FPGA modules

### Integration Tests
- End-to-end decoding pipeline
- Host-FPGA communication
- Performance regression tests
- Accuracy verification vs reference

### Human Verification Tests
- Decode known test patterns
- Verify resource estimates
- Measure actual latency
- Confirm adaptive behavior

## Verification and Validation

- Correctness: Compare predictions with reference decoder
- Performance: Measure latency and throughput
- Resource: Verify FPGA resource usage within estimates
- Accuracy: Maintain decoding error rate within target
- Adaptation: Verify strategy adaptation behavior

## References

- Vegapunk: Accurate and Fast Decoding for Quantum LDPC Codes, MICRO 2025
- Quantum LDPC Codes: Theory and Practice
- FPGA-based LDPC Decoders: Survey and Trends
