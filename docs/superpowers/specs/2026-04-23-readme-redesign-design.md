# README Redesign Design Spec

**Date**: 2026-04-23
**Status**: Approved
**Author**: Claude Code
**Repository**: 0420-FPGA-Decoder

## Overview

This document defines the redesign of the root `README.md` for the FPGA decoder exploration workspace. The redesign adopts a **state-driven modular architecture** to serve a mixed audience of developers, researchers, and visitors, with emphasis on clear project status and roadmap visibility.

## Design Goals

1. **State Visibility**: Make project status and roadmap the primary navigation anchor
2. **Modular Navigation**: Enable readers to quickly jump to relevant sections based on their needs
3. **Mixed Audience Support**: Balance technical depth for developers with accessibility for researchers
4. **Maintainability**: Structure that evolves naturally as project phases complete

## Proposed Structure

### 1. Header

```markdown
# 0420-FPGA-Decoder

[![CI](...)](...)  [![License](...)](...)  [![Docs](...)](...)

Exploration workspace for FPGA quantum decoders, including Surface Code and QLDPC Python implementations with Verilog interface validation.
```

### 2. Project Status Dashboard (Core Navigation)

A table showing current status of all major modules:

| Module | Status | Description |
|--------|--------|-------------|
| 🔷 Surface Code Decoder | ✅ Production Ready | Python benchmark + PyMWPM + Verilog interface validation complete |
| 🟨 QLDPC Decoder | 🔨 Phase 2 in Development | Adaptive sparse architecture, strategy engine implemented, integrating |
| 🟦 Python-Verilog Interface | ✅ Validation Complete | Interface-level contract model with automated CI verification |
| ⬜ GPU Decoder | 🔮 Planned | Parked, future consideration |

**Rationale**: This is the primary navigation anchor. Status is the common baseline all audiences care about.

### 3. Quick Navigation

Card-style links to key resources:

- 🚀 **[Quick Start](#quick-start)** - Run verification in 5 minutes
- 📚 **[Documentation](../../README.md)** - Complete architecture and design docs
- 🧪 **[Verification](../../verification/surface-code-verilog.md)** - CI and manual verification
- 📈 **[Roadmap](#roadmap)** - Development plans and milestones

### 4. Quick Start

#### Environment Setup

```bash
conda create -n fpga-decoder python=3.11
conda activate fpga-decoder
cd decoder
pip install -e ".[dev]"
```

#### Installation Verification (30 seconds)

```bash
cd decoder
python -m pytest tests/test_verilog_cli.py -q
```

#### Full Verification Flow (5 minutes)

```bash
# 1. Generate Verilog vectors
python -m decoder_benchmark.verilog_cli \
  --distance 3 --error-rate 0.001 --shots 4 \
  --rounds 3 --seed 1234 \
  --output-dir ../verilog_test/vectors/current

# 2. Run Verilog interface simulation (requires iverilog)
iverilog -o ../verilog_test/sim.out \
  ../verilog_test/decoder_interface_model.v \
  ../verilog_test/tb_decoder_interface.v
vvp ../verilog_test/sim.out
```

> 💡 **Tip**: Complete flow and troubleshooting in [Surface Code Verilog Verification](../../verification/surface-code-verilog.md)

**Rationale**: Progressive complexity - quick validation first, then full flow. Links to detailed docs for edge cases.

### 5. Project Modules

Each module follows this structure:

```markdown
### 🔷 decoder/ — Surface Code Decoder

Python implementation of rotated Surface Code benchmark:
- `circuit_factory.py` - Stim-based circuit generation
- `syndrome.py` - Detector event sampling
- `decoder.py` - PyMatching MWPM reference
- `verilog_vectors.py` - Verilog-friendly export
- `notebooks/` - Interactive Jupyter notebook

**Status**: ✅ Production Ready
**Docs**: [Surface Code Architecture](../../architecture/surface-code-decoder.md)
```

Module descriptions are concise (3-5 lines) with key files and links to detailed docs.

### 6. Roadmap

#### Current Focus (2026 Q2)

- [ ] **QLDPC Phase 2 Complete** - Integrate adaptive strategy engine into main flow
- [ ] **QLDPC End-to-End Verification** - Establish complete test benchmarks and performance reports
- [ ] **Jupyter Demo** - QLDPC decoder interactive demonstration

#### Near-Term (2026 Q3-Q4)

- [ ] **QLDPC Phase 3** - FPGA hardware architecture design
- [ ] **Performance Optimization** - Establish benchmarks for Surface Code and QLDPC
- [ ] **Documentation Polish** - Add API docs and more usage examples

#### Long-Term Vision

- [ ] **GPU Decoder** - Evaluate and potentially restart GPU implementation path
- [ ] **Multi-Decoder Integration** - Unified interface supporting multiple decoding strategies
- [ ] **Open Source Release** - Clean dependencies, prepare for broader release

**Rationale**: Clear timeframes with specific deliverables. Checkboxes for tracking progress.

### 7. Contributing

#### Reporting Issues

- Report bugs or feature requests in GitHub Issues
- Include environment info, reproduction steps, expected behavior

#### Submitting Code

1. Fork repository and create feature branch
2. Follow Follow CLAUDE.md (at repository root) development guidelines development guidelines
3. Ensure all tests pass
4. Run documentation link check: `python scripts/check_docs_links.py`
5. Submit Pull Request

#### Code Standards

- Keep commit messages clear and concise
- Add necessary test cases
- Update relevant documentation
- Follow existing code style

### 8. License

[To be determined/specific license type]

## Design Principles

1. **Status as Anchor**: The status dashboard is the first substantive content after the header
2. **Progressive Disclosure**: Quick validation first, full flow later
3. **Link-Down Strategy**: Detailed information lives in `docs/`, README provides navigation
4. **Visual Hierarchy**: Emojis, tables, and code blocks create scannable structure
5. **Minimal Friction**: Every command in Quick Start is copy-paste ready

## Implementation Notes

- Preserve existing CI badge
- Maintain PowerShell/Windows compatibility in code examples
- Keep existing `docs/` link structure
- Add license badge when license is determined
- Status emojis use consistent color coding:
  - 🔷 Blue = completed/stable
  - 🟨 Yellow = in progress
  - 🟦 Light blue = supporting
  - ⬜ White = planned/future
  - ✅ = done
  - 🔨 = building
  - 🔮 = future

## Success Criteria

- New visitors understand project value and status within 10 seconds
- Developers can run verification in under 5 minutes
- Researchers can find relevant architecture docs in 3 clicks
- Status updates require changing only one table row
- Document structure scales as new modules are added
