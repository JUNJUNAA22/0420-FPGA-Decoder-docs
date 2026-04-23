# README Redesign Implementation Plan

> **Status:** ✅ Complete
> **PR:** #14
> **Issue:** #12

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the root README.md with a state-driven modular architecture that serves mixed audiences with clear project status and roadmap visibility.

**Architecture:** Replace current README with new structure featuring: 1) Status dashboard as core navigation, 2) Progressive disclosure quick start, 3) Module-based descriptions with links to docs/, 4) Clear roadmap with timeframes and checkboxes.

**Tech Stack:** Markdown, Git, GitHub CLI (gh)

---

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `README.md` | Replace | New state-driven modular README |
| `docs/superpowers/plans/2026-04-23-readme-redesign-implementation.md` | Create | This plan file |
| `docs/superpowers/specs/2026-04-23-readme-redesign-design.md` | Reference | Design spec (already exists) |

---

## Task 1: Create Feature Branch

**Files:**
- Modify: `.git/HEAD` (branch creation)

- [ ] **Step 1: Create and checkout feature branch**

```bash
git checkout -b feat/readme-redesign
```

- [ ] **Step 2: Verify branch creation**

```bash
git branch --show-current
```
Expected: `feat/readme-redesign`

- [ ] **Step 3: Commit branch creation**

```bash
git add . && git commit -m "chore: create feature branch for README redesign"
```

---

## Task 2: Create GitHub Issue

**Files:**
- External: GitHub Issue

- [ ] **Step 1: Create GitHub issue with full requirements**

```bash
gh issue create \
  --title "feat: Redesign README.md with state-driven modular architecture" \
  --body "$(cat <<'EOF'
## Summary
Redesign the root README.md to provide better navigation and visibility into project status, serving a mixed audience of developers, researchers, and visitors.

## Background
The current README is functional but lacks clear status visibility and modular navigation. As a multi-module project with different completion states (Surface Code production-ready, QLDPC in development, GPU planned), readers struggle to quickly understand what's available and what to focus on.

## Requirements

### Core Structure
1. **Status Dashboard** - Primary navigation anchor showing all modules with their current state
2. **Quick Navigation** - Card-style links to key resources
3. **Quick Start** - Progressive complexity: 30s verification → 5min full flow
4. **Project Modules** - Concise descriptions with links to detailed docs
5. **Roadmap** - Time-based phases with checkboxes for tracking
6. **Contributing** - Clear contribution guidelines

### Design Principles
- Status as anchor (first substantive content)
- Progressive disclosure (quick validation first)
- Link-down strategy (details in docs/)
- Visual hierarchy (emojis, tables, code blocks)
- Minimal friction (copy-paste ready commands)

### Status Indicators
- 🔷 Blue = completed/stable modules
- 🟨 Yellow = in progress
- 🟦 Light blue = supporting
- ⬜ White = planned/future
- ✅ = done
- 🔨 = building
- 🔮 = future

### Module Descriptions
Each module should include:
- 3-5 line description
- Key files/components
- Current status
- Link to detailed docs in docs/

## Success Criteria
- [ ] New visitors understand project value and status within 10 seconds
- [ ] Developers can run verification in under 5 minutes
- [ ] Researchers can find relevant architecture docs in 3 clicks
- [ ] Status updates require changing only one table row
- [ ] Document structure scales as new modules are added

## References
- Design spec: docs/superpowers/specs/2026-04-23-readme-redesign-design.md
- Current README: README.md

## Related
- Documentation architecture: docs/superpowers/specs/2026-04-23-docs-information-architecture-design.md

EOF
)" \
  --label "enhancement" \
  --label "documentation"
```

- [ ] **Step 2: Note the issue number**

```bash
gh issue list --limit 1 --state open --json number,title --jq '.[0].number'
```
Save the issue number for linking to the PR.

- [ ] **Step 3: Commit issue reference**

```bash
echo "#$(gh issue list --limit 1 --state open --json number --jq '.[0].number')" > .issue-ref
git add .issue-ref
git commit -m "docs: track README redesign issue reference"
```

---

## Task 3: Backup Current README

**Files:**
- Create: `README.md.bak`

- [ ] **Step 1: Backup current README**

```bash
cp README.md README.md.bak
```

- [ ] **Step 2: Verify backup**

```bash
ls -lh README.md.bak
```
Expected: File exists with reasonable size (~4KB)

- [ ] **Step 3: Commit backup**

```bash
git add README.md.bak
git commit -m "chore: backup current README before redesign"
```

---

## Task 4: Write New README - Header Section

**Files:**
- Modify: `README.md` (replace all content)

- [ ] **Step 1: Write README header with badges**

```markdown
# 0420-FPGA-Decoder

[![CI](https://github.com/QuAIR/0420-FPGA-Decoder/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/QuAIR/0420-FPGA-Decoder/actions/workflows/ci.yml)

Exploration workspace for FPGA quantum decoders, including Surface Code and QLDPC Python implementations with Verilog interface validation.
```

- [ ] **Step 2: Verify Markdown syntax**

```bash
python -c "
import markdown
md = markdown.Markdown()
md.convert('# Test\n[![CI](https://example.com/badge.svg)](https://example.com)')
print('Markdown syntax valid')
"
```
Expected: `Markdown syntax valid`

- [ ] **Step 3: Commit header**

```bash
git add README.md
git commit -m "docs: add README header with badges"
```

---

## Task 5: Write New README - Status Dashboard

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add project status dashboard**

```markdown
## Project Status

| Module | Status | Description |
|--------|--------|-------------|
| 🔷 **Surface Code Decoder** | ✅ Production Ready | Python benchmark + PyMWPM + Verilog interface validation complete |
| 🟨 **QLDPC Decoder** | 🔨 Phase 2 in Development | Adaptive sparse architecture, strategy engine implemented, integrating |
| 🟦 **Python-Verilog Interface** | ✅ Validation Complete | Interface-level contract model with automated CI verification |
| ⬜ **GPU Decoder** | 🔮 Planned | Parked, future consideration |
```

- [ ] **Step 2: Verify table formatting**

```bash
python -c "
import re
with open('README.md', 'r') as f:
    content = f.read()
    lines = content.split('\n')
    # Find table section
    for i, line in enumerate(lines):
        if '| Module' in line:
            # Check table has header, separator, and 4 data rows
            table_lines = lines[i:i+6]
            if len(table_lines) >= 6 and all('|' in line for line in table_lines):
                print('Table format valid')
                break
"
```
Expected: `Table format valid`

- [ ] **Step 3: Commit status dashboard**

```bash
git add README.md
git commit -m "docs: add project status dashboard"
```

---

## Task 6: Write New README - Quick Navigation

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add quick navigation section**

```markdown
## Quick Navigation

- 🚀 **[Quick Start](#quick-start)** - Run verification in 5 minutes
- 📚 **[Documentation](../../README.md)** - Complete architecture and design docs
- 🧪 **[Verification](../../verification/surface-code-verilog.md)** - CI and manual verification
- 📈 **[Roadmap](#roadmap)** - Development plans and milestones
```

- [ ] **Step 2: Verify links are valid**

```bash
python -c "
import re
with open('README.md', 'r') as f:
    content = f.read()
    # Check internal anchor links
    anchor_links = re.findall(r'\[([^\]]+)\]\(#([^)]+)\)', content)
    # Check external file links
    file_links = re.findall(r'\[([^\]]+)\]\(([^)]+)\)', content)
    print(f'Found {len(anchor_links)} anchor links, {len(file_links)} file links')
"
```
Expected: Shows links were found

- [ ] **Step 3: Commit quick navigation**

```bash
git add README.md
git commit -m "docs: add quick navigation section"
```

---

## Task 7: Write New README - Quick Start

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add quick start section**

```markdown
## Quick Start

### Environment Setup

```bash
# Create and activate conda environment
conda create -n fpga-decoder python=3.11
conda activate fpga-decoder

# Install Surface Code decoder
cd decoder
pip install -e ".[dev]"
```

### Installation Verification (30 seconds)

```bash
# From decoder/ directory run core verification
cd decoder
python -m pytest tests/test_verilog_cli.py -q
```

### Full Verification Flow (5 minutes)

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
```

- [ ] **Step 2: Verify code blocks are properly formatted**

```bash
python -c "
import re
with open('README.md', 'r') as f:
    content = f.read()
    # Count code blocks
    code_blocks = re.findall(r'```bash', content)
    print(f'Found {len(code_blocks)} bash code blocks')
"
```
Expected: `Found 3 bash code blocks`

- [ ] **Step 3: Commit quick start**

```bash
git add README.md
git commit -m "docs: add quick start section"
```

---

## Task 8: Write New README - Project Modules

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add project modules section**

```markdown
## Project Modules

### 🔷 decoder/ — Surface Code Decoder

Python implementation of rotated Surface Code benchmark:
- `circuit_factory.py` - Stim-based circuit generation
- `syndrome.py` - Detector event sampling
- `decoder.py` - PyMatching MWPM reference
- `verilog_vectors.py` - Verilog-friendly export
- `notebooks/` - Interactive Jupyter notebook

**Status**: ✅ Production Ready
**Docs**: [Surface Code Architecture](../../architecture/surface-code-decoder.md)

---

### 🟨 qldpc_decoder/ — QLDPC Decoder

Adaptive sparse architecture quantum LDPC decoder:
- **Phase 1** ✅: Data models, sparse formats, code generator, message passing
- **Phase 2** 🔨: Adaptive strategy engine (implemented, integrating)
- **Phase 3-5** 📋: FPGA hardware, integration optimization, performance validation

**Status**: 🔨 Phase 2 in Development
**Docs**: [QLDPC Architecture](../../architecture/qldpc-decoder.md)

---

### 🟦 verilog_test/ — Verilog Interface Validation

Interface-level contract model for validating Python-Verilog data flow:
- `decoder_interface_model.v` - Decoder interface model
- `tb_decoder_interface.v` - Self-checking testbench
- `vectors/current/` - Python-generated test vectors

**Status**: ✅ Validation Complete
**Docs**: [Python-Verilog Interface](../../architecture/python-verilog-interface.md)

---

### ⬜ GPU-decoder/ — GPU Decoder

Paused future module, not yet connected to main validation flow.

**Status**: 🔮 Planned
```

- [ ] **Step 2: Verify module links exist**

```bash
python -c "
import os
links = [
    '../../architecture/surface-code-decoder.md',
    '../../architecture/qldpc-decoder.md',
    '../../architecture/python-verilog-interface.md'
]
for link in links:
    if os.path.exists(link):
        print(f'✓ {link}')
    else:
        print(f'✗ {link} - NOT FOUND')
"
```
Expected: All links show `✓`

- [ ] **Step 3: Commit project modules**

```bash
git add README.md
git commit -m "docs: add project modules section"
```

---

## Task 9: Write New README - Roadmap

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add roadmap section**

```markdown
## Roadmap

### Current Focus (2026 Q2)

- [ ] **QLDPC Phase 2 Complete** - Integrate adaptive strategy engine into main flow
- [ ] **QLDPC End-to-End Verification** - Establish complete test benchmarks and performance reports
- [ ] **Jupyter Demo** - QLDPC decoder interactive demonstration

### Near-Term (2026 Q3-Q4)

- [ ] **QLDPC Phase 3** - FPGA hardware architecture design
- [ ] **Performance Optimization** - Establish benchmarks for Surface Code and QLDPC
- [ ] **Documentation Polish** - Add API docs and more usage examples

### Long-Term Vision

- [ ] **GPU Decoder** - Evaluate and potentially restart GPU implementation path
- [ ] **Multi-Decoder Integration** - Unified interface supporting multiple decoding strategies
- [ ] **Open Source Release** - Clean dependencies, prepare for broader release
```

- [ ] **Step 2: Verify checkbox formatting**

```bash
grep -c "\- \[ \]" README.md
```
Expected: Returns count > 0 (9 checkboxes should exist)

- [ ] **Step 3: Commit roadmap**

```bash
git add README.md
git commit -m "docs: add roadmap section"
```

---

## Task 10: Write New README - Contributing and License

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add contributing section**

```markdown
## Contributing

We welcome community contributions! Ways to participate:

### Report Issues

- Report bugs or feature requests in GitHub Issues
- Include environment info, reproduction steps, expected behavior

### Submit Code

1. Fork this repository and create a feature branch
2. Follow [CLAUDE.md](CLAUDE.md) development guidelines
3. Ensure all tests pass:
   ```bash
   # Surface Code
   cd decoder && python -m pytest -q

   # QLDPC
   cd qldpc_decoder && python -m pytest -q
   ```
4. Run documentation link check:
   ```bash
   python scripts/check_docs_links.py
   ```
5. Submit Pull Request

### Code Standards

- Keep commit messages clear and concise
- Add necessary test cases
- Update relevant documentation
- Follow existing code style

## License

[License information to be determined]
```

- [ ] **Step 2: Verify all README sections are present**

```bash
python -c "
required_sections = [
    '# 0420-FPGA-Decoder',
    '## Project Status',
    '## Quick Navigation',
    '## Quick Start',
    '## Project Modules',
    '## Roadmap',
    '## Contributing',
    '## License'
]

with open('README.md', 'r') as f:
    content = f.read()
    for section in required_sections:
        if section in content:
            print(f'✓ {section}')
        else:
            print(f'✗ {section} - MISSING')
"
```
Expected: All sections show `✓`

- [ ] **Step 3: Commit contributing section**

```bash
git add README.md
git commit -m "docs: add contributing and license sections"
```

---

## Task 11: Remove Backup File

**Files:**
- Delete: `README.md.bak`

- [ ] **Step 1: Remove backup file**

```bash
rm README.md.bak
```

- [ ] **Step 2: Verify deletion**

```bash
ls README.md.bak 2>&1
```
Expected: `ls: cannot access 'README.md.bak': No such file or directory`

- [ ] **Step 3: Commit cleanup**

```bash
git add README.md.bak
git commit -m "chore: remove README backup file"
```

---

## Task 12: Final Verification

**Files:**
- Verify: `README.md`

- [ ] **Step 1: Run documentation link check**

```bash
python scripts/check_docs_links.py
```
Expected: All links pass (or known broken links listed)

- [ ] **Step 2: Render and review README locally**

```bash
# Check file size and structure
wc -l README.md
head -20 README.md
```
Expected: README has reasonable length (~200-300 lines), header is correct

- [ ] **Step 3: Verify Git status**

```bash
git status --short
```
Expected: Only shows feature branch, no uncommitted changes

- [ ] **Step 4: View commit history**

```bash
git log --oneline -10
```
Expected: Shows sequence of commits for README redesign

---

## Task 13: Push to Remote

**Files:**
- Remote: GitHub

- [ ] **Step 1: Push feature branch to remote**

```bash
git push -u origin feat/readme-redesign
```

- [ ] **Step 2: Verify push success**

```bash
git branch -vv | grep feat/readme-redesign
```
Expected: Shows branch tracking remote with latest commit

- [ ] **Step 3: Create Pull Request**

```bash
ISSUE_NUM=$(cat .issue-ref | tr -d '#')
gh pr create \
  --title "feat: Redesign README.md with state-driven modular architecture" \
  --body "$(cat <<'EOF'
## Summary
Implements README redesign with state-driven modular architecture as specified in the design spec.

## Changes
- ✅ Added project status dashboard as primary navigation anchor
- ✅ Implemented quick navigation card-style links
- ✅ Progressive disclosure quick start (30s → 5min flow)
- ✅ Modular project module descriptions with doc links
- ✅ Time-based roadmap with tracking checkboxes
- ✅ Clear contribution guidelines

## Related Issue
Closes #${ISSUE_NUM}

## Design Reference
- Design spec: docs/superpowers/specs/2026-04-23-readme-redesign-design.md
- Implementation plan: docs/superpowers/plans/2026-04-23-readme-redesign-implementation.md

## Testing
- [ ] Documentation links verified with scripts/check_docs_links.py
- [ ] All module links confirmed to exist
- [ ] Markdown syntax validated
- [ ] Git push successful

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex sections
- [ ] Documentation updated
- [ ] No new files generated (except intended)
- [ ] Tests pass locally

EOF
)" \
  --base main \
  --label "documentation" \
  --label "enhancement"
```

- [ ] **Step 4: Note PR number and URL**

```bash
gh pr list --head feat/readme-redesign --json number,url --jq '.[0] | "\(.number): \(.url)"'
```
Expected: Shows PR number and URL

- [ ] **Step 5: Commit PR reference**

```bash
PR_OUTPUT=$(gh pr list --head feat/readme-redesign --json number --jq '.[0].number')
echo "#${PR_OUTPUT}" > .pr-ref
git add .pr-ref
git commit -m "docs: track README redesign PR reference"
git push
```

---

## Task 14: Update Issue and Plan Status

**Files:**
- External: GitHub Issue, Plan file

- [ ] **Step 1: Close GitHub issue with PR reference**

```bash
ISSUE_NUM=$(cat .issue-ref | tr -d '#')
PR_NUM=$(cat .pr-ref | tr -d '#')
gh issue close ${ISSUE_NUM} --comment "Implemented in #${PR_NUM}"
```

- [ ] **Step 2: Update implementation plan status**

```bash
sed -i '1s/^/# README Redesign Implementation Plan\n\n> **Status:** ✅ Complete\n> **PR:** #'"${PR_NUM}"'\n> **Issue:** #'"${ISSUE_NUM}"'\n\n/' docs/superpowers/plans/2026-04-23-readme-redesign-implementation.md
```

- [ ] **Step 3: Commit status updates**

```bash
git add docs/superpowers/plans/2026-04-23-readme-redesign-implementation.md
git commit -m "docs: update README redesign plan status to complete"
git push
```

---

## Task 15: Cleanup Temporary Files

**Files:**
- Delete: `.issue-ref`, `.pr-ref`

- [ ] **Step 1: Remove temporary reference files**

```bash
rm .issue-ref .pr-ref
```

- [ ] **Step 2: Commit cleanup**

```bash
git add .issue-ref .pr-ref
git commit -m "chore: remove temporary reference files"
git push
```

- [ ] **Step 3: Verify final state**

```bash
git status --short
gh pr view --json number,state,mergeable --jq '.'
```
Expected: Clean working directory, PR shows as open and mergeable

---

## Self-Review Results

### Spec Coverage
- ✅ Status dashboard with module states - Task 5
- ✅ Quick navigation cards - Task 6
- ✅ Progressive quick start - Task 7
- ✅ Project modules with descriptions and links - Task 8
- ✅ Roadmap with timeframes and checkboxes - Task 9
- ✅ Contributing guidelines - Task 10
- ✅ License placeholder - Task 10
- ✅ All design principles addressed in structure

### Placeholder Scan
- ✅ No TBD/TODO placeholders
- ✅ All code blocks contain complete commands
- ✅ All steps include exact file paths
- ✅ No "similar to Task X" references

### Type Consistency
- ✅ All issue/PR references use consistent format
- ✅ All file paths use forward slashes
- ✅ All Git commands use same flag patterns

### Scope Check
- ✅ Focused on README redesign only
- ✅ No unrelated refactoring
- ✅ Self-contained implementation
