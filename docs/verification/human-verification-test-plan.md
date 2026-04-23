# Human Verification Test Plan for QLDPC Adaptive Sparse Decoder

## Overview

This issue tracks human verification test plan for QLDPC Adaptive Hybrid Sparse Decoder implementation. The goal is to validate that implementation meets design requirements for real-time decoding with millisecond-level latency.

## Test Objectives

1. **Correctness**: Verify decoding accuracy against reference implementations
2. **Performance**: Confirm millisecond-level latency and target throughput
3. **Resource**: Validate FPGA resource usage within estimates
4. **Adaptation**: Verify adaptive sparse strategy behavior
5. **Integration**: Test end-to-end pipeline functionality

## Test Plan

### Phase 1: Code Generator Verification

#### Test 1.1: Code Structure Validation
- [ ] Generate random QLDPC codes (n=100, 500, 1000)
- [ ] Verify parity check matrices (Hx, Hz) are valid
- [ ] Check sparsity pattern meets random QLDPC criteria
- [ ] Verify no zero rows or columns

#### Test 1.2: Code Properties
- [ ] Calculate and verify code rate
- [ ] Check minimum distance (if computationally feasible)
- [ ] Verify Tanner graph properties (no self-loops, no multi-edges if required)
- [ ] Validate node degree distribution

### Phase 2: Adaptive Strategy Engine Verification

#### Test 2.1: Code Feature Analysis
- [ ] Test with different code structures (sparse vs dense, regular vs irregular)
- [ ] Verify sparsity pattern analysis correctness
- [ ] Check node degree distribution calculation
- [ ] Validate cycle structure detection

#### Test 2.2: Strategy Selection
- [ ] Test strategy selection under different resource budgets
- [ ] Verify strategy changes based on code features
- [ ] Check strategy adaptation during decoding iterations
- [ ] Validate convergence detection logic

### Phase 3: FPGA Simulation Verification

#### Test 3.1: Individual Module Tests
- [ ] Verify SMPU (Sparse Message Passing Unit) functionality
- [ ] Verify PCNP (Parallel Check Node Processor) functionality
- [ ] Verify SMSE (Sparse Matrix Storage Engine) functionality
- [ ] Verify FPGA controller data flow

#### Test 3.2: Integration Tests
- [ ] Test complete FPGA pipeline with synthetic data
- [ ] Verify streaming data flow between modules
- [ ] Check memory access patterns and BRAM usage
- [ ] Validate clock domain crossing (if applicable)

### Phase 4: End-to-End Verification

#### Test 4.1: Correctness Tests
- [ ] Decode known test patterns and verify against ground truth
- [ ] Compare results with reference decoder (e.g., PyMatching for surface code baseline)
- [ ] Test with various syndrome patterns (all-zeros, single errors, multiple errors)
- [ ] Verify logical error rate within expected range

#### Test 4.2: Performance Tests
- [ ] Measure actual latency with different code sizes (n=100, 500, 1000)
- [ ] Measure throughput (shots/second)
- [ ] Verify latency is < 1.5ms (target) or < 1ms (stretch goal)
- [ ] Profile performance bottlenecks

#### Test 4.3: Resource Tests
- [ ] Synthesize FPGA design and measure actual resource usage
- [ ] Compare against resource estimates (LUT, BRAM, DSP)
- [ ] Verify timing closure meets target frequency
- [ ] Check power consumption

#### Test 4.4: Adaptive Behavior Tests
- [ ] Verify strategy engine selects different strategies for different codes
- [ ] Observe and validate strategy changes during decoding iterations
- [ ] Test with resource constraints to verify fallback behavior
- [ ] Validate that adaptive decisions improve performance

### Phase 5: Integration with Existing Decoder

#### Test 5.1: API Compatibility
- [ ] Verify QLDPC decoder API matches Surface Code decoder interface
- [ ] Test switching between decoders in benchmark code
- [ ] Verify output format compatibility
- [ ] Check performance monitoring integration

#### Test 5.2: Benchmarking
- [ ] Run benchmark suite with QLDPC decoder
- [ ] Compare performance against Surface Code baseline
- [ ] Generate performance reports
- [ ] Verify regression detection

## Test Data

### Required Test Vectors
- [ ] Small code test cases (n=100)
- [ ] Medium code test cases (n=500)
- [ ] Large code test cases (n=1000)
- [ ] Known error patterns for correctness validation
- [ ] Stress test patterns (high error rates)

### Reference Implementations
- [ ] Python reference decoder for correctness comparison
- [ ] Industry-standard decoder for validation (if available)

## Success Criteria

### Correctness
- [ ] Decoding accuracy within 5% of reference implementation
- [ ] No incorrect decoding on test patterns with known solutions
- [ ] Logical error rate matches theoretical expectations

### Performance
- [ ] Latency < 1.5ms for n=1000
- [ ] Throughput > 1000 shots/second
- [ ] Performance consistent across multiple runs

### Resource
- [ ] LUT usage within 65-90% of Alveo U250
- [ ] BRAM usage within 55-80%
- [ ] DSP usage within 35-55%
- [ ] Timing closure at target frequency

### Adaptation
- [ ] Strategy engine makes different decisions for different codes
- [ ] Adaptation improves performance vs non-adaptive baseline
- [ ] No unexpected strategy changes during decoding

## Open Questions

1. Which FPGA board will be used for final verification?
2. What is acceptable tolerance for decoding accuracy?
3. What error rate range should be tested?
4. How many test cases are sufficient for each phase?

## Notes

- All tests should be automated where possible
- Document any deviations from expected results
- Include performance profiling data
- Save synthesis reports for resource verification

---

**References:**
- [QLDPC Adaptive Sparse Decoder Design](../specs/qldpc/2026-04-22-qldpc-adaptive-sparse-decoder-design.md)
- [Vegapunk: Accurate and Fast Decoding for Quantum LDPC Codes](https://dl.acm.org/doi/full/10.1145/3725843.3756084)

---

**To create this as a GitHub issue:**

1. Run `gh auth login` to authenticate with GitHub
2. Run: `gh issue create --title "Human Verification Test Plan for QLDPC Adaptive Sparse Decoder" --body-file HUMAN_VERIFICATION_TEST_PLAN.md`
