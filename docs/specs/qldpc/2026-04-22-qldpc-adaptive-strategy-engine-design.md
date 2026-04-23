# QLDPC Adaptive Strategy Engine Design

## Goal

Implement the Adaptive Strategy Engine (Phase 2) for the QLDPC decoder, enabling intelligent sparse strategy selection and runtime adaptation based on code characteristics and decoding progress.

## Scope

This design covers:
- Code feature analysis: sparsity, node degree distribution, cycle structure, connected components
- Strategy selection logic: optimal sparse strategy based on code features and resource budget
- Resource estimation: simulation-level estimation (no runtime monitoring)
- Strategy adaptation: dynamic adjustment during decoding iterations (stuck detection, convergence optimization)

This design does not cover:
- Real-time resource monitoring (reserved for Phase 3/4)
- FPGA hardware implementation (Phase 3)
- Host-FPGA communication (Phase 4)
- Performance optimization (Phase 5)

## Architecture

The Adaptive Strategy Engine consists of three primary components:

### 1. CodeAnalyzer

Analyzes QLDPC parity check matrices to extract structural features:

- **Basic metrics**: Sparsity, average/max/min node degree
- **Cycle structure**: BFS-based detection of short cycles (length ≤ 6)
- **Connected components**: BFS-based component counting and size analysis

```python
class CodeAnalyzer:
    max_cycle_length: int = 6  # Maximum cycle length to detect

    def analyze(self, h: np.ndarray) -> CodeFeatures
    def _analyze_basic_metrics(self, h: np.ndarray) -> tuple[float, float, int, int]
    def _analyze_cycles(self, h: np.ndarray) -> tuple[int, float]
    def _analyze_components(self, h: np.ndarray) -> tuple[int, int]
```

### 2. StrategySelector

Selects optimal sparse strategy based on code features and FPGA resource budget:

- **Graph pruning level**: Based on sparsity and code complexity
- **Zero-value threshold**: Based on cycle complexity and node degree
- **Compression format**: Delta, RLE, or none based on code size and resources
- **Parallelization degree**: Based on component size and DSP availability

```python
class StrategySelector:
    def select_strategy(self, features: CodeFeatures, budget: ResourceBudget) -> SparseStrategy
    def _select_pruning_level(self, features, budget) -> int
    def _select_threshold(self, features, budget) -> float
    def _select_compression(self, features, budget) -> str
    def _select_parallelization(self, features, budget) -> int
```

### 3. StrategyAdapter

Adapts strategy during decoding iterations based on progress:

- **Stuck detection**: Identifies when decoding is stuck (no progress)
- **Convergence detection**: Identifies when decoder has converged
- **Unstuck strategy**: Increases pruning level and threshold when stuck
- **Convergence strategy**: Enables early stop with minimal pruning when converged

```python
class StrategyAdapter:
    convergence_window: int = 5
    stuck_threshold: int = 3
    adaptation_interval: int = 10

    def should_adapt(self, progress: DecodingProgress) -> bool
    def adapt(self, progress: DecodingProgress) -> StrategyUpdate
    def _detect_stuck(self, progress: DecodingProgress) -> bool
    def _detect_convergence(self, progress: DecodingProgress) -> bool
```

### 4. AdaptiveStrategyEngine

Main engine coordinating analysis, selection, and adaptation:

```python
class AdaptiveStrategyEngine:
    analyzer: CodeAnalyzer
    selector: StrategySelector
    adapter: StrategyAdapter

    def initialize_strategy(self, h_x, h_z, config) -> tuple[SparseStrategy, SparseStrategy]
    def should_adapt(self, progress: DecodingProgress) -> bool
    def adapt(self, progress: DecodingProgress) -> StrategyUpdate
```

## Data Models

### Extended CodeFeatures

```python
@dataclass
class CodeFeatures:
    # Existing fields
    sparsity: float
    avg_node_degree: float
    cycle_count: int
    min_distance: int

    # New fields for Phase 2
    max_node_degree: int
    min_node_degree: int
    avg_cycle_length: float
    n_connected_components: int
    max_component_size: int
```

### SparseStrategy

```python
@dataclass
class SparseStrategy:
    graph_pruning_level: int        # 0=none, 1=light, 2=heavy
    zero_threshold: float           # Zero-value filter threshold (0-1)
    compression_format: str         # "none" | "delta" | "rle"
    parallelization_degree: int       # 1=serial, 2/4/8=parallel
```

### DecodingProgress

```python
@dataclass
class DecodingProgress:
    iteration: int
    convergence_rate: float      # Message change rate
    stuck_iterations: int       # Consecutive iterations with no change
    last_syndrome_weight: int # Last syndrome weight
```

### StrategyUpdate

```python
@dataclass
class StrategyUpdate:
    new_pruning_level: Optional[int] = None
    new_threshold: Optional[float] = None
    convergence_early_stop: bool = False
```

## Data Flow

```
Initialization:
  QLDPC matrices (Hx, Hz)
    -> CodeAnalyzer.analyze()
    -> CodeFeatures (sparsity, cycles, components, etc.)
    -> StrategySelector.select_strategy(features, budget)
    -> SparseStrategy (pruning, threshold, compression, parallelization)

During Decoding:
  Each iteration
    -> Track DecodingProgress (iteration, convergence, stuck count)
    -> StrategyAdapter.should_adapt()
    -> If yes: StrategyAdapter.adapt()
    -> StrategyUpdate (new pruning/threshold or early stop)
    -> Apply update to decoder (Phase 3)
```

## Algorithm Details

### Cycle Detection

Uses BFS to find cycles in Tanner graph:

- Start from each variable node
- Traverse: variable -> check -> variable paths
- Cycle detected when start == end and path length ≥ 4 (even)
- Maximum cycle length: 6 (configurable)

Time complexity: O(V × E) where V = variable nodes, E = edges
Space complexity: O(V) for visited tracking

### Connected Component Analysis

Uses BFS to count and size connected components:

- Mark visited nodes during traversal
- Start new BFS from each unvisited node
- Track component size for maximum detection

Time complexity: O(V + E)
Space complexity: O(V)

### Strategy Selection Heuristics

**Pruning Level:**
- High sparsity (>0.8) → No pruning
- Many components (>3) → Heavy pruning
- Short cycles (<4 length) → Light pruning
- Resource constraint (<60% LUT) → Reduce pruning level

**Zero Threshold:**
- Base: 0.01 (1%)
- Many cycles → Increase to 0.015
- Low node degree (<3) → Decrease to 0.008
- Upper bound: 0.1 (10%)

**Compression Format:**
- Large code (>500 vars) + multi-component → Delta encoding
- High sparsity (>0.7) + high BRAM (>70%) → RLE
- Otherwise → None

**Parallelization:**
- Small component (<200 nodes) → Serial (1)
- High DSP (>50%) → High parallel (8)
- Medium DSP (>30%) → Medium parallel (4)
- Default → Basic parallel (2)

### Adaptation Triggers

**Stuck Detection:**
- Consecutive iterations with convergence_rate > 0.001
- Threshold: 3 iterations

**Convergence Detection:**
- convergence_rate < 0.001

**Adaptation Interval:**
- Check every 10 iterations
- Can be configured

## Error Handling

### Input Validation

- Matrix dimensions: Must be 2D
- Matrix values: Must be 0 or 1
- Sparsity: Must be in [0, 1]
- Budget values: Must be in [0, 100] for percentages

### Boundary Cases

- Empty matrices: Raise ValueError
- Disconnected components: Handle gracefully (analyze each component)
- No cycles found: Return 0 for cycle metrics
- Single component: max_component_size = total nodes

## Testing Strategy

### Unit Tests

1. **CodeAnalyzer tests** (3+ tests):
   - Basic metrics (sparsity, degree distribution)
   - Cycle detection with known graphs
   - Connected component counting

2. **StrategySelector tests** (2+ tests):
   - Consistency across different code features
   - Resource budget impact on decisions

3. **StrategyAdapter tests** (2+ tests):
   - Stuck detection and unstuck strategy
   - Convergence detection and early stop

4. **Integration test** (1 test):
   - Full engine initialization and strategy selection

### Integration Points

- Existing `message_passing.py` will consume `SparseStrategy` parameters (Phase 3)
- Existing `QLDPCDecoder` will integrate `AdaptiveStrategyEngine`
- No breaking changes to Phase 1 implementation

## File Structure Updates

```
qldpc_decoder/src/qldpc/
├── adaptive_strategy.py        # New file: all new classes
│   ├── CodeAnalyzer
│   ├── StrategySelector
│   ├── StrategyAdapter
│   └── AdaptiveStrategyEngine
│
├── models.py                 # Extend: add 4 new fields to CodeFeatures
│                           # Add: SparseStrategy, DecodingProgress, StrategyUpdate
│
└── decoder.py               # Modify: integrate AdaptiveStrategyEngine

qldpc_decoder/tests/
└── test_adaptive_strategy.py  # New file: 8+ tests
```

## Dependencies

- NumPy (existing)
- Existing `models.py` (CodeFeatures, ResourceBudget)
- Existing `message_passing.py` (will consume strategies in Phase 3)

No new external dependencies required.

## Performance Considerations

### Complexity

- **Code analysis**: O(V × E) for each matrix (once per initialization)
- **Strategy selection**: O(1) - table lookup and simple heuristics
- **Adaptation check**: O(1) - simple threshold comparison
- **Overall impact**: Negligible (< 1% of decoding time)

### Memory

- Code features: ~100 bytes
- Strategy object: ~50 bytes
- Progress tracking: ~50 bytes
- Total overhead: < 1KB per decoder instance

## Future Extensions

Reserved for Phase 3/4:
- Real-time resource monitoring hooks
- Hardware-specific strategy application
- Adaptive parameters tuning from performance data
- Machine learning-based strategy selection

## References

- Design parent: `2026-04-22-qldpc-adaptive-sparse-decoder-design.md`
- Phase 1 implementation: `2026-04-22-qldpc-decoder-phase1-implementation.md`
