# QLDPC Decoder Jupyter Demo Design

## Goal

Create an interactive Jupyter notebook demonstrating the complete QLDPC decoder pipeline with comprehensive visualizations for educational and demonstration purposes.

## Scope

This demo covers:

- QLDPC code generation with configurable parameters
- Syndrome sampling at multiple error rates
- Adaptive strategy selection and execution
- Five visualization types: confusion matrix, accuracy vs error rate, convergence plot, sparsity pattern, strategy adaptation timeline
- End-to-end decoding pipeline demonstration

This demo does not cover:

- FPGA hardware execution (Python simulation only)
- Large-scale benchmarking (uses small code sizes for quick demo)
- Advanced decoding strategies beyond what's implemented in Phase 2
- Integration with Surface Code decoder comparison

## Notebook Structure

### Section 1: Setup and Imports
- Import dependencies: numpy, matplotlib, seaborn, networkx, qldpc
- Configure matplotlib/seaborn styles for consistent visuals
- Define reusable helper functions for visualizations

### Section 2: QLDPC Code Generation
- Generate random QLDPC code (n=150, k=80 for quick execution)
- Display code properties: dimensions, rate, sparsity
- Visualize sparsity pattern of H_x and H_z matrices (Visualization #4)

### Section 3: Syndrome Sampling
- Define error rates to test: [0.01, 0.03, 0.05, 0.10]
- Sample 100 syndromes per error rate
- Display example syndrome patterns
- Show error rate distribution histogram

### Section 4: Decoding with Adaptive Strategy
- Initialize QLDPCDecoder with AdaptiveStrategyEngine
- Run decode_batch on all syndromes
- Track DecodingProgress for each error rate
- Monitor strategy adaptations during decoding

### Section 5: Results Visualization

**Visualization #1: Confusion Matrix**
- Compare predictions vs true errors
- Show accuracy percentage
- Highlight common failure patterns

**Visualization #2: Decoding Accuracy vs Error Rate**
- Line plot: error rate (x-axis) vs accuracy (y-axis)
- Show accuracy degradation trend
- Mark threshold where accuracy drops below 90%

**Visualization #3: Convergence Plot**
- Multi-line plot: error rate per iteration (y-axis) vs iteration (x-axis)
- Separate lines for different initial error rates
- Show convergence speed and final residual error

**Visualization #4: Sparsity Pattern** (already shown in Section 2)
- Two subplot imshow plots for H_x and H_z
- Use colormap to highlight non-zero entries
- Display sparsity percentage

**Visualization #5: Strategy Adaptation Timeline**
- Timeline plot showing when adaptations occurred
- Display what changed (pruning level, threshold, compression, parallelization)
- Correlate adaptations with convergence events

**Summary Statistics Table**
- Overall accuracy across all error rates
- Average iterations to convergence
- Number of strategy adaptations
- Execution time

## Dependencies

```python
numpy >= 1.24
matplotlib >= 3.7
seaborn >= 0.12
networkx >= 3.1
qldpc (local package)
jupyter >= 1.0
ipykernel >= 6.0
```

## File Location

`qldpc_decoder/demo/qldpc_decoder_demo.ipynb`

## Execution Time

Target: < 2 minutes for full notebook execution on typical laptop
- Code generation: ~2s
- Syndrome sampling: ~3s
- Decoding: ~60s
- Visualization: ~5s

## Output Artifacts

The notebook generates:
- Inline visualizations displayed in Jupyter
- Optional: save plots as PNG files to `demo/output/`

## Testing

Verify the demo works by:
1. Running all cells sequentially
2. Confirming all visualizations render without errors
3. Checking that accuracy metrics are reasonable (>90% at low error rates)
4. Verifying strategy adaptations are visible in timeline
