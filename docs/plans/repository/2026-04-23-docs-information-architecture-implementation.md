<!-- docs-link-check: allow-old-paths -->

# Repository Documentation Information Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reorganize repository documentation for issue #6 into a clear top-level `docs/` information architecture while preserving compatibility stubs at old paths.

**Architecture:** Keep source, Verilog, CI, and package directories in place. Move long-term documentation into topic-based `docs/` sections, replace old documentation paths with short stubs, and add a local Markdown link checker for verification.

**Tech Stack:** Markdown, Python 3 standard library, Git on Windows PowerShell.

---

## Scope Check

This plan implements `docs/superpowers/specs/2026-04-23-docs-information-architecture-design.md`. It is one documentation migration, not a code-module layout change. Do not move `surface_code_decoder/`, `verilog_test/`, `qldpc_decoder/`, `GPU-decoder/`, `.github/`, or runtime test directories.

## File Structure

Create long-term documentation directories:

- `docs/README.md`: documentation navigation entry point.
- `docs/architecture/overview.md`: current repository architecture summary.
- `docs/architecture/surface-code-decoder.md`: current `surface_code_decoder/` Surface Code benchmark summary.
- `docs/architecture/python-verilog-interface.md`: current Python-to-Verilog interface and Verilog contract model summary.
- `docs/architecture/qldpc-decoder.md`: conservative current QLDPC module status.
- `docs/architecture/gpu-decoder.md`: parked future module status.
- `docs/development/environment.md`: setup and command entry points.
- `docs/development/generated-artifacts.md`: generated artifact policy.
- `docs/verification/ci.md`: CI workflow and local equivalents.
- `docs/verification/surface-code-verilog.md`: vector generation and Verilog self-check.
- `docs/verification/human-verification-test-plan.md`: moved human verification plan.
- `docs/archive/README.md`: archive policy.
- `docs/specs/repository/`: issue #6 documentation architecture spec.
- `docs/specs/surface-code/`: Surface Code design specs.
- `docs/specs/interface/`: Python-Verilog interface design specs.
- `docs/specs/qldpc/`: QLDPC design specs.
- `docs/plans/repository/`: issue #6 implementation plan.
- `docs/plans/surface-code/`: Surface Code implementation plans.
- `docs/plans/interface/`: Python-Verilog interface implementation plans.
- `docs/plans/qldpc/`: QLDPC implementation plans.

Retain compatibility stubs at old paths:

- `docs/superpowers/specs/*.md`
- `docs/superpowers/plans/*.md`
- `surface_code_decoder/docs/superpowers/specs/*.md`
- `surface_code_decoder/docs/superpowers/plans/*.md`
- `GPU-decoder/docs/superpowers/specs/*.md`
- `GPU-decoder/docs/superpowers/plans/*.md`
- `HUMAN_VERIFICATION_TEST_PLAN.md`

Add verification script:

- `scripts/check_docs_links.py`: repository-local Markdown link and stale path checker.

Modify entry-point docs:

- `README.md`: slim project entry point.
- `surface_code_decoder/README.md`: Surface Code subproject entry point with new docs links.
- `qldpc_decoder/README.md`: conservative status table.
- `AGENTS.md`: updated agent guidance and new docs links.
- `CLAUDE.md`: updated agent guidance and new docs links.

## Task 1: Add The Markdown Link Checker

**Files:**
- Create: `scripts/check_docs_links.py`

- [ ] **Step 1: Create the checker script**

Create `scripts/check_docs_links.py` with this content:

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import re
import sys
from dataclasses import dataclass
from pathlib import Path
from urllib.parse import unquote, urlparse


LINK_RE = re.compile(r"(?<!!)\[[^\]]+\]\(([^)]+)\)")
ALLOW_OLD_PATHS_MARKER = "<!-- docs-link-check: allow-old-paths -->"

OLD_DOC_PREFIXES = (
    "docs/superpowers/",
    "surface_code_decoder/docs/superpowers/",
    "GPU-decoder/docs/superpowers/",
)

COMPATIBILITY_STUB_DIRS = (
    Path("docs/superpowers"),
    Path("surface_code_decoder/docs/superpowers"),
    Path("GPU-decoder/docs/superpowers"),
)

COMPATIBILITY_STUB_FILES = {
    Path("HUMAN_VERIFICATION_TEST_PLAN.md"),
}

SKIP_DIR_NAMES = {
    ".git",
    ".mypy_cache",
    ".pytest_cache",
    ".ruff_cache",
    ".venv",
    "__pycache__",
    "benchmark-out",
    "node_modules",
    "pytest-temp-runs",
}

SKIP_DIR_PREFIXES = (
    ".pytest-basetemp",
    "pytest-basetemp",
)


@dataclass(frozen=True)
class Finding:
    path: Path
    line: int
    message: str


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Check repository-local Markdown links.")
    parser.add_argument(
        "--root",
        type=Path,
        default=Path.cwd(),
        help="Repository root to scan. Defaults to the current directory.",
    )
    return parser.parse_args()


def main() -> int:
    args = parse_args()
    root = args.root.resolve()
    findings: list[Finding] = []

    for markdown_path in iter_markdown_files(root):
        rel_path = markdown_path.relative_to(root)
        text = markdown_path.read_text(encoding="utf-8")
        findings.extend(check_local_links(root, markdown_path, rel_path, text))
        findings.extend(check_stale_doc_paths(rel_path, text))

    if findings:
        for finding in findings:
            print(f"{finding.path}:{finding.line}: {finding.message}")
        return 1

    print("Documentation links OK.")
    return 0


def iter_markdown_files(root: Path) -> list[Path]:
    return sorted(
        path
        for path in root.rglob("*.md")
        if path.is_file() and not should_skip(path.relative_to(root))
    )


def should_skip(rel_path: Path) -> bool:
    for part in rel_path.parts:
        if part in SKIP_DIR_NAMES:
            return True
        if any(part.startswith(prefix) for prefix in SKIP_DIR_PREFIXES):
            return True
    return False


def check_local_links(root: Path, markdown_path: Path, rel_path: Path, text: str) -> list[Finding]:
    findings: list[Finding] = []
    for match in LINK_RE.finditer(text):
        raw_target = match.group(1).strip()
        line = text.count("\n", 0, match.start()) + 1
        target = normalize_markdown_target(raw_target)

        if should_ignore_target(target):
            continue

        target_path = (markdown_path.parent / target).resolve()
        try:
            target_path.relative_to(root)
        except ValueError:
            findings.append(Finding(rel_path, line, f"local link escapes repository: {raw_target}"))
            continue

        if not target_path.exists():
            findings.append(Finding(rel_path, line, f"missing local link target: {raw_target}"))

    return findings


def normalize_markdown_target(raw_target: str) -> str:
    target = raw_target
    if " " in target and not target.startswith("<"):
        target = target.split()[0]
    target = target.strip("<>")
    target = target.split("#", 1)[0]
    target = target.split("?", 1)[0]
    return unquote(target)


def should_ignore_target(target: str) -> bool:
    if not target:
        return True
    parsed = urlparse(target)
    if parsed.scheme in {"http", "https", "mailto"}:
        return True
    if target.startswith("#"):
        return True
    return False


def check_stale_doc_paths(rel_path: Path, text: str) -> list[Finding]:
    if stale_paths_allowed(rel_path, text):
        return []

    findings: list[Finding] = []
    for old_prefix in OLD_DOC_PREFIXES:
        search_start = 0
        while True:
            index = text.find(old_prefix, search_start)
            if index == -1:
                break
            line = text.count("\n", 0, index) + 1
            findings.append(
                Finding(
                    rel_path,
                    line,
                    f"stale documentation path reference: {old_prefix}",
                )
            )
            search_start = index + len(old_prefix)
    return findings


def stale_paths_allowed(rel_path: Path, text: str) -> bool:
    if ALLOW_OLD_PATHS_MARKER in text:
        return True
    if rel_path in COMPATIBILITY_STUB_FILES:
        return True
    return any(is_relative_to(rel_path, stub_dir) for stub_dir in COMPATIBILITY_STUB_DIRS)


def is_relative_to(path: Path, parent: Path) -> bool:
    try:
        path.relative_to(parent)
    except ValueError:
        return False
    return True


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Run the checker before migration**

Run from the repository root:

```powershell
python scripts\check_docs_links.py
```

Expected: the command may fail before old references are migrated. Record whether it reports stale `docs/superpowers/`, `surface_code_decoder/docs/superpowers/`, or `GPU-decoder/docs/superpowers/` references. Do not change the script to hide real findings.

- [ ] **Step 3: Commit the checker**

```powershell
git add scripts\check_docs_links.py
git commit -m "docs: add markdown link checker"
```

## Task 2: Migrate Specs And Plans Into Topic Directories

**Files:**
- Move: `docs/superpowers/specs/2026-04-23-docs-information-architecture-design.md` to `docs/specs/repository/2026-04-23-docs-information-architecture-design.md`
- Move: `docs/superpowers/plans/2026-04-23-docs-information-architecture-implementation.md` to `docs/plans/repository/2026-04-23-docs-information-architecture-implementation.md`
- Move: `surface_code_decoder/docs/superpowers/specs/2026-04-20-surface-code-realtime-mwpm-fpga-design.md` to `docs/specs/surface-code/2026-04-20-surface-code-realtime-mwpm-fpga-design.md`
- Move: `surface_code_decoder/docs/superpowers/plans/2026-04-20-surface-code-realtime-mwpm-fpga.md` to `docs/plans/surface-code/2026-04-20-surface-code-realtime-mwpm-fpga.md`
- Move: `GPU-decoder/docs/superpowers/specs/2026-04-20-python-decoder-verilog-interface-design.md` to `docs/specs/interface/2026-04-20-python-decoder-verilog-interface-design.md`
- Move: `GPU-decoder/docs/superpowers/plans/2026-04-20-python-decoder-verilog-interface-implementation.md` to `docs/plans/interface/2026-04-20-python-decoder-verilog-interface-implementation.md`
- Move: `GPU-decoder/docs/superpowers/plans/2026-04-20-verilog-counter-smoke-test.md` to `docs/plans/interface/2026-04-20-verilog-counter-smoke-test.md`
- Move: `docs/superpowers/specs/2026-04-22-qldpc-adaptive-sparse-decoder-design.md` to `docs/specs/qldpc/2026-04-22-qldpc-adaptive-sparse-decoder-design.md`
- Move: `docs/superpowers/specs/2026-04-22-qldpc-adaptive-strategy-engine-design.md` to `docs/specs/qldpc/2026-04-22-qldpc-adaptive-strategy-engine-design.md`
- Move: `docs/superpowers/specs/2026-04-22-qldpc-jupyter-demo-design.md` to `docs/specs/qldpc/2026-04-22-qldpc-jupyter-demo-design.md`
- Move: `docs/superpowers/plans/2026-04-22-qldpc-decoder-phase1-implementation.md` to `docs/plans/qldpc/2026-04-22-qldpc-decoder-phase1-implementation.md`
- Move: `docs/superpowers/plans/2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md` to `docs/plans/qldpc/2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md`
- Modify: every old moved path listed above, replacing it with a compatibility stub.

- [ ] **Step 1: Create target directories**

```powershell
New-Item -ItemType Directory -Force docs\specs\repository,docs\specs\surface-code,docs\specs\interface,docs\specs\qldpc,docs\plans\repository,docs\plans\surface-code,docs\plans\interface,docs\plans\qldpc
```

- [ ] **Step 2: Move the files with Git**

Run these exact commands from the repository root:

```powershell
git mv docs\superpowers\specs\2026-04-23-docs-information-architecture-design.md docs\specs\repository\2026-04-23-docs-information-architecture-design.md
git mv docs\superpowers\plans\2026-04-23-docs-information-architecture-implementation.md docs\plans\repository\2026-04-23-docs-information-architecture-implementation.md
git mv surface_code_decoder\docs\superpowers\specs\2026-04-20-surface-code-realtime-mwpm-fpga-design.md docs\specs\surface-code\2026-04-20-surface-code-realtime-mwpm-fpga-design.md
git mv surface_code_decoder\docs\superpowers\plans\2026-04-20-surface-code-realtime-mwpm-fpga.md docs\plans\surface-code\2026-04-20-surface-code-realtime-mwpm-fpga.md
git mv GPU-decoder\docs\superpowers\specs\2026-04-20-python-decoder-verilog-interface-design.md docs\specs\interface\2026-04-20-python-decoder-verilog-interface-design.md
git mv GPU-decoder\docs\superpowers\plans\2026-04-20-python-decoder-verilog-interface-implementation.md docs\plans\interface\2026-04-20-python-decoder-verilog-interface-implementation.md
git mv GPU-decoder\docs\superpowers\plans\2026-04-20-verilog-counter-smoke-test.md docs\plans\interface\2026-04-20-verilog-counter-smoke-test.md
git mv docs\superpowers\specs\2026-04-22-qldpc-adaptive-sparse-decoder-design.md docs\specs\qldpc\2026-04-22-qldpc-adaptive-sparse-decoder-design.md
git mv docs\superpowers\specs\2026-04-22-qldpc-adaptive-strategy-engine-design.md docs\specs\qldpc\2026-04-22-qldpc-adaptive-strategy-engine-design.md
git mv docs\superpowers\specs\2026-04-22-qldpc-jupyter-demo-design.md docs\specs\qldpc\2026-04-22-qldpc-jupyter-demo-design.md
git mv docs\superpowers\plans\2026-04-22-qldpc-decoder-phase1-implementation.md docs\plans\qldpc\2026-04-22-qldpc-decoder-phase1-implementation.md
git mv docs\superpowers\plans\2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md docs\plans\qldpc\2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md
```

- [ ] **Step 3: Add historical status notes to moved historical plans**

For each moved plan under `docs/plans/surface-code/`, `docs/plans/interface/`, and `docs/plans/qldpc/`, insert this note immediately after the title line:

```md
> Status: Historical implementation plan. The work described here may already
> be partially or fully implemented. Use the corresponding architecture page and
> current source tree as the source of truth.
```

Do not add that status note to `docs/plans/repository/2026-04-23-docs-information-architecture-implementation.md` while it is the active implementation plan.

- [ ] **Step 4: Add the allow marker to repository migration docs**

Ensure these files contain this marker as their first line:

```md
<!-- docs-link-check: allow-old-paths -->
```

Files:

- `docs/specs/repository/2026-04-23-docs-information-architecture-design.md`
- `docs/plans/repository/2026-04-23-docs-information-architecture-implementation.md`

- [ ] **Step 5: Recreate compatibility stubs at old paths**

Each old path must contain only a title, moved target, and removal condition. Use this exact pattern, substituting the title and target:

```md
# Original Document Title

This document moved to:

`relative/path/from/old/location/to/new/document.md`

This compatibility stub can be removed after downstream references, open
branches, and agent guidance have been verified against the new path.
```

Use these titles and target paths:

| Old path | Title | Target |
| --- | --- | --- |
| `docs/superpowers/specs/2026-04-23-docs-information-architecture-design.md` | `Repository Documentation Information Architecture Design` | `../../specs/repository/2026-04-23-docs-information-architecture-design.md` |
| `docs/superpowers/plans/2026-04-23-docs-information-architecture-implementation.md` | `Repository Documentation Information Architecture Implementation Plan` | `../../plans/repository/2026-04-23-docs-information-architecture-implementation.md` |
| `surface_code_decoder/docs/superpowers/specs/2026-04-20-surface-code-realtime-mwpm-fpga-design.md` | `Surface Code Realtime MWPM FPGA Design` | `../../../../docs/specs/surface-code/2026-04-20-surface-code-realtime-mwpm-fpga-design.md` |
| `surface_code_decoder/docs/superpowers/plans/2026-04-20-surface-code-realtime-mwpm-fpga.md` | `Surface Code Realtime MWPM FPGA Implementation Plan` | `../../../../docs/plans/surface-code/2026-04-20-surface-code-realtime-mwpm-fpga.md` |
| `GPU-decoder/docs/superpowers/specs/2026-04-20-python-decoder-verilog-interface-design.md` | `Python Decoder To Verilog Interface Simulation Design` | `../../../../docs/specs/interface/2026-04-20-python-decoder-verilog-interface-design.md` |
| `GPU-decoder/docs/superpowers/plans/2026-04-20-python-decoder-verilog-interface-implementation.md` | `Python Decoder To Verilog Interface Implementation Plan` | `../../../../docs/plans/interface/2026-04-20-python-decoder-verilog-interface-implementation.md` |
| `GPU-decoder/docs/superpowers/plans/2026-04-20-verilog-counter-smoke-test.md` | `Verilog Counter Smoke Test Plan` | `../../../../docs/plans/interface/2026-04-20-verilog-counter-smoke-test.md` |
| `docs/superpowers/specs/2026-04-22-qldpc-adaptive-sparse-decoder-design.md` | `QLDPC Adaptive Sparse Decoder Design` | `../../specs/qldpc/2026-04-22-qldpc-adaptive-sparse-decoder-design.md` |
| `docs/superpowers/specs/2026-04-22-qldpc-adaptive-strategy-engine-design.md` | `QLDPC Adaptive Strategy Engine Design` | `../../specs/qldpc/2026-04-22-qldpc-adaptive-strategy-engine-design.md` |
| `docs/superpowers/specs/2026-04-22-qldpc-jupyter-demo-design.md` | `QLDPC Jupyter Demo Design` | `../../specs/qldpc/2026-04-22-qldpc-jupyter-demo-design.md` |
| `docs/superpowers/plans/2026-04-22-qldpc-decoder-phase1-implementation.md` | `QLDPC Decoder Phase 1 Implementation Plan` | `../../plans/qldpc/2026-04-22-qldpc-decoder-phase1-implementation.md` |
| `docs/superpowers/plans/2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md` | `QLDPC Phase 2 Adaptive Strategy Engine Implementation Plan` | `../../plans/qldpc/2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md` |

- [ ] **Step 6: Verify this migration slice**

Run:

```powershell
python scripts\check_docs_links.py
git status --short
```

Expected: the link checker may still report stale references from root README, agent guidance, and subproject README files. It must not report missing targets for the moved spec/plan stubs.

- [ ] **Step 7: Commit the migration slice**

```powershell
git add docs surface_code_decoder\docs GPU-decoder\docs
git commit -m "docs: migrate specs and plans into topic directories"
```

## Task 3: Add Navigation, Architecture, Development, And Verification Pages

**Files:**
- Create: `docs/README.md`
- Create: `docs/architecture/overview.md`
- Create: `docs/architecture/surface-code-decoder.md`
- Create: `docs/architecture/python-verilog-interface.md`
- Create: `docs/architecture/qldpc-decoder.md`
- Create: `docs/architecture/gpu-decoder.md`
- Create: `docs/development/environment.md`
- Create: `docs/development/generated-artifacts.md`
- Create: `docs/verification/ci.md`
- Create: `docs/verification/surface-code-verilog.md`
- Create: `docs/archive/README.md`
- Move: `HUMAN_VERIFICATION_TEST_PLAN.md` to `docs/verification/human-verification-test-plan.md`
- Recreate: `HUMAN_VERIFICATION_TEST_PLAN.md` as compatibility stub.

- [ ] **Step 1: Create documentation directories**

```powershell
New-Item -ItemType Directory -Force docs\architecture,docs\development,docs\verification,docs\archive
```

- [ ] **Step 2: Move the human verification plan**

```powershell
git mv HUMAN_VERIFICATION_TEST_PLAN.md docs\verification\human-verification-test-plan.md
```

Create root `HUMAN_VERIFICATION_TEST_PLAN.md` with:

```md
# Human Verification Test Plan

This document moved to:

`docs/verification/human-verification-test-plan.md`

This compatibility stub can be removed after downstream references, open
branches, and agent guidance have been verified against the new path.
```

- [ ] **Step 3: Create `docs/README.md`**

Write `docs/README.md` with these sections:

```md
# Documentation

This directory is the long-term documentation entry point for the FPGA decoder
exploration workspace.

## Current Architecture

- [Repository Overview](architecture/overview.md)
- [Surface Code Decoder](architecture/surface-code-decoder.md)
- [Python-Verilog Interface](architecture/python-verilog-interface.md)
- [QLDPC Decoder](architecture/qldpc-decoder.md)
- [GPU Decoder](architecture/gpu-decoder.md)

## Development

- [Environment](development/environment.md)
- [Generated Artifacts](development/generated-artifacts.md)

## Verification

- [CI](verification/ci.md)
- [Surface Code Verilog Self-Check](verification/surface-code-verilog.md)
- [Human Verification Test Plan](verification/human-verification-test-plan.md)

## Design Specifications

- [Repository Documentation](specs/repository/2026-04-23-docs-information-architecture-design.md)
- [Surface Code](specs/surface-code/2026-04-20-surface-code-realtime-mwpm-fpga-design.md)
- [Python-Verilog Interface](specs/interface/2026-04-20-python-decoder-verilog-interface-design.md)
- [QLDPC Adaptive Sparse Decoder](specs/qldpc/2026-04-22-qldpc-adaptive-sparse-decoder-design.md)
- [QLDPC Adaptive Strategy Engine](specs/qldpc/2026-04-22-qldpc-adaptive-strategy-engine-design.md)
- [QLDPC Jupyter Demo](specs/qldpc/2026-04-22-qldpc-jupyter-demo-design.md)

## Implementation Plans

Plans preserve implementation history and handoff details. Historical plans are
not the current source of truth; use architecture pages and source files for
current behavior.

- [Repository Documentation Migration](plans/repository/2026-04-23-docs-information-architecture-implementation.md)
- [Surface Code Realtime MWPM FPGA](plans/surface-code/2026-04-20-surface-code-realtime-mwpm-fpga.md)
- [Python-Verilog Interface](plans/interface/2026-04-20-python-decoder-verilog-interface-implementation.md)
- [Verilog Counter Smoke Test](plans/interface/2026-04-20-verilog-counter-smoke-test.md)
- [QLDPC Decoder Phase 1](plans/qldpc/2026-04-22-qldpc-decoder-phase1-implementation.md)
- [QLDPC Adaptive Strategy Phase 2](plans/qldpc/2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md)

## Archive

- [Archive Policy](archive/README.md)
```

- [ ] **Step 4: Create architecture pages**

Create each architecture page using these required facts:

`docs/architecture/overview.md`:

- State that this repository is an FPGA quantum decoder exploration workspace.
- List current root-level modules: `surface_code_decoder/`, `verilog_test/`, `qldpc_decoder/`, and `GPU-decoder/`.
- State that current root-level locations are present state, not a permanent layout constraint.
- Link to each module-specific architecture page.

`docs/architecture/surface-code-decoder.md`:

- Describe `surface_code_decoder/` as the Surface Code Python benchmark package.
- Mention circuit generation, syndrome sampling, PyMatching/MWPM decoding, latency modeling, reporting, and Verilog vector export.
- Link to `../specs/surface-code/2026-04-20-surface-code-realtime-mwpm-fpga-design.md`.

`docs/architecture/python-verilog-interface.md`:

- Describe Python vector generation from `surface_code_decoder/`.
- Describe Verilog consumption through `verilog_test/vectors/current/`.
- State that `verilog_test/decoder_interface_model.v` is an interface contract model, not a hardware MWPM decoder.
- Link to `../specs/interface/2026-04-20-python-decoder-verilog-interface-design.md` and `../verification/surface-code-verilog.md`.

`docs/architecture/qldpc-decoder.md`:

- Describe `qldpc_decoder/` as an adaptive sparse QLDPC decoder module.
- Use a conservative status table with rows for Python package scaffold, models, sparse format, message passing, decoder API, adaptive strategy engine, code generator, FPGA host interface, Verilog FPGA modules, integration tests, and performance validation.
- Mark current implementation as implemented, partial, skeleton, or planned based on files in `qldpc_decoder/src/` and skipped tests in `qldpc_decoder/tests/test_integration.py`.

`docs/architecture/gpu-decoder.md`:

- Describe `GPU-decoder/` as a parked future module owned separately.
- State that it is not part of the current Python-Verilog interface validation path.
- State that future integration should be handled by a dedicated design and implementation plan.

`docs/archive/README.md`:

- Define archive as obsolete but still useful design drafts, plans, or verification notes.
- State that generated reports, runtime outputs, and current source-of-truth docs do not belong in archive.

- [ ] **Step 5: Create development and verification pages**

`docs/development/environment.md` must include:

- Python 3.11 or newer.
- Conda environment name `fpga-decoder`.
- Editable install command from `surface_code_decoder/`.
- Verilog simulator requirement for `iverilog` and `vvp`.
- Note that Windows PowerShell profile execution-policy warnings are not project failures when commands still run and exit correctly.

`docs/development/generated-artifacts.md` must include:

- `surface_code_decoder/benchmark-out/`
- `verilog_test/vectors/current/`
- pytest temp directories under `pytest-temp-runs/` and `.pytest-basetemp*`
- simulator outputs such as `.out`, `.vcd`, and `sim.out`
- Python caches such as `__pycache__/`
- State that these are regenerated, not moved into long-term docs.

`docs/verification/ci.md` must include:

- `.github/workflows/ci.yml`
- `python-tests` job summary.
- `verilog-interface` job summary.
- Local equivalents should point to `surface-code-verilog.md` for vector and Verilog commands.

`docs/verification/surface-code-verilog.md` must include these commands:

```powershell
conda run -n fpga-decoder python -m pytest tests/test_verilog_vectors.py tests/test_verilog_cli.py -q --basetemp ..\pytest-temp-runs\task1\run-local
conda run -n fpga-decoder python -m decoder_benchmark.verilog_cli --distance 3 --error-rate 0.001 --shots 4 --rounds 3 --seed 1234 --output-dir ..\verilog_test\vectors\current
iverilog -o verilog_test\sim_decoder_interface.out verilog_test\decoder_interface_model.v verilog_test\tb_decoder_interface.v
vvp verilog_test\sim_decoder_interface.out
```

Also include the fallback pytest command when `surface_code_decoder/tests/test_verilog_vectors.py` is absent.

- [ ] **Step 6: Verify this documentation slice**

Run:

```powershell
python scripts\check_docs_links.py
```

Expected: missing-link findings should be fixed before committing. Stale path findings may remain from root README, `AGENTS.md`, `CLAUDE.md`, `surface_code_decoder/README.md`, or `qldpc_decoder/README.md` until Task 4.

- [ ] **Step 7: Commit this slice**

```powershell
git add docs HUMAN_VERIFICATION_TEST_PLAN.md
git commit -m "docs: add documentation navigation and current architecture pages"
```

## Task 4: Update Entry Points And Agent Guidance

**Files:**
- Modify: `README.md`
- Modify: `surface_code_decoder/README.md`
- Modify: `qldpc_decoder/README.md`
- Modify: `AGENTS.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Slim root `README.md`**

Rewrite root `README.md` as a concise project entry point with these sections:

- `# 0420-FPGA-Decoder`
- `## Purpose`
- `## Core Modules`
- `## Documentation`
- `## Quick Start`
- `## Generated Files`
- `## Contributing Notes`

Required module bullets:

- `surface_code_decoder/`: Surface Code Python benchmark and vector exporter.
- `verilog_test/`: Verilog interface-level self-check harness.
- `qldpc_decoder/`: QLDPC adaptive sparse decoder module.
- `GPU-decoder/`: parked future module, not connected to current validation flow.

Required docs links:

- `docs/README.md`
- `docs/development/environment.md`
- `docs/verification/surface-code-verilog.md`
- `docs/development/generated-artifacts.md`

- [ ] **Step 2: Update `surface_code_decoder/README.md`**

Keep it as a Surface Code subproject entry point. Ensure it links to:

- `../docs/architecture/surface-code-decoder.md`
- `../docs/architecture/python-verilog-interface.md`
- `../docs/verification/surface-code-verilog.md`
- `../docs/development/generated-artifacts.md`

Retain enough local quick-start detail for someone working directly in `surface_code_decoder/`.

- [ ] **Step 3: Update `qldpc_decoder/README.md`**

Replace the old implementation checklist with a conservative status table:

```md
## Implementation Status

| Area | Status | Notes |
| --- | --- | --- |
| Package scaffold | Implemented | `pyproject.toml`, `src/qldpc/`, and tests are present. |
| Models | Implemented | Core dataclasses live in `src/qldpc/models.py`. |
| Sparse format | Implemented | Sparse matrix helpers live in `src/qldpc/sparse_format.py`. |
| Message passing | Implemented | The current decoder core lives in `src/qldpc/message_passing.py`. |
| Decoder API | Partial | `QLDPCDecoder` exists, but `decode_batch()` currently uses a simplified `H_x` path. |
| Adaptive strategy engine | Partial | Code analysis and strategy selection exist, with future runtime adaptation still expected. |
| Code generator | Planned | `src/qldpc/code_generator.py` exists, but integration tests still mark generation work as not complete. |
| FPGA host interface | Skeleton | `src/qldpc/fpga_interface.py` is present for future host integration. |
| Verilog FPGA modules | Skeleton | `src/fpga/` contains module skeletons that are not yet the current validation path. |
| End-to-end integration tests | Planned | `tests/test_integration.py` still contains skipped tests for the full pipeline. |
| Performance validation | Planned | Performance regression tests are not yet active. |
```

Update the design links to:

- `../docs/architecture/qldpc-decoder.md`
- `../docs/specs/qldpc/2026-04-22-qldpc-adaptive-sparse-decoder-design.md`
- `../docs/specs/qldpc/2026-04-22-qldpc-adaptive-strategy-engine-design.md`
- `../docs/plans/qldpc/2026-04-22-qldpc-decoder-phase1-implementation.md`
- `../docs/plans/qldpc/2026-04-22-qldpc-phase2-adaptive-strategy-engine-implementation.md`

- [ ] **Step 4: Update `AGENTS.md`**

Keep agent rules, but update important paths to the new structure:

- Replace old Python-Verilog interface design path with `docs/specs/interface/2026-04-20-python-decoder-verilog-interface-design.md`.
- Replace old Python-Verilog interface plan path with `docs/plans/interface/2026-04-20-python-decoder-verilog-interface-implementation.md`.
- Reference `docs/verification/surface-code-verilog.md` for vector generation and Verilog simulation details.
- Reference `docs/development/environment.md` for environment setup.
- Reference `docs/development/generated-artifacts.md` for generated file policy.
- Keep warnings about Windows pytest temp directories and PowerShell profile warnings.

- [ ] **Step 5: Update `CLAUDE.md`**

Mirror the same stable documentation path changes made in `AGENTS.md`. Keep Claude-specific guidance if present, and avoid introducing contradictions with `AGENTS.md`.

- [ ] **Step 6: Check stale path references**

Run:

```powershell
rg "docs/superpowers|surface_code_decoder/docs/superpowers|GPU-decoder/docs/superpowers" README.md AGENTS.md CLAUDE.md surface_code_decoder\README.md qldpc_decoder\README.md docs
```

Expected: remaining matches should be compatibility stubs or repository migration docs containing `<!-- docs-link-check: allow-old-paths -->`.

- [ ] **Step 7: Run link checker**

```powershell
python scripts\check_docs_links.py
```

Expected: `Documentation links OK.`

- [ ] **Step 8: Commit entry-point updates**

```powershell
git add README.md surface_code_decoder\README.md qldpc_decoder\README.md AGENTS.md CLAUDE.md docs
git commit -m "docs: update repository entry points for new docs structure"
```

## Task 5: Final Documentation Verification

**Files:**
- Verify: all Markdown files under the repository, excluding generated and temporary directories.

- [ ] **Step 1: Run the link checker**

```powershell
python scripts\check_docs_links.py
```

Expected:

```text
Documentation links OK.
```

- [ ] **Step 2: Check for stale old-path references**

```powershell
rg "docs/superpowers|surface_code_decoder/docs/superpowers|GPU-decoder/docs/superpowers" -g "*.md" -g "!**/benchmark-out/**" -g "!**/pytest-temp-runs/**" -g "!**/.pytest-basetemp*/**"
```

Expected: matches are limited to compatibility stubs and repository migration docs that contain the allow marker.

- [ ] **Step 3: Check for generated files in the diff**

```powershell
git status --short
```

Expected: no generated artifacts from these paths should be staged or modified:

- `surface_code_decoder/benchmark-out/`
- `verilog_test/vectors/current/`
- `pytest-temp-runs/`
- `.pytest-basetemp*`
- `__pycache__/`
- simulator outputs such as `.out` or `.vcd`

- [ ] **Step 4: Check whitespace**

```powershell
git diff --check
```

Expected: no output and exit code 0.

- [ ] **Step 5: Commit verification-only fixes if any were required**

If Steps 1 through 4 required doc-only fixes, commit them:

```powershell
git add README.md AGENTS.md CLAUDE.md docs surface_code_decoder\README.md qldpc_decoder\README.md scripts\check_docs_links.py HUMAN_VERIFICATION_TEST_PLAN.md
git commit -m "docs: fix documentation links after migration"
```

If no fixes were required, do not create an empty commit.

## Task 6: Handoff Summary

**Files:**
- Verify: Git history and working tree.

- [ ] **Step 1: Show final status**

```powershell
git status --short --branch
```

Expected: clean working tree on `codex/issue-6-docs-architecture`, except for known local permission warnings from pytest temp directories.

- [ ] **Step 2: Summarize commits**

```powershell
git log --oneline --decorate -n 8
```

Expected: recent commits include:

- `docs: add markdown link checker`
- `docs: migrate specs and plans into topic directories`
- `docs: add documentation navigation and current architecture pages`
- `docs: update repository entry points for new docs structure`
- optional `docs: fix documentation links after migration`

- [ ] **Step 3: Prepare final response**

Report:

- the new `docs/` structure;
- that no source module directories were moved;
- that compatibility stubs remain at old paths;
- that `scripts/check_docs_links.py` passed;
- any remaining local-only warnings, especially Windows locked pytest temp directories.
