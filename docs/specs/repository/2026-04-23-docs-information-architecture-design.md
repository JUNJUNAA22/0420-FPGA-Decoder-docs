<!-- docs-link-check: allow-old-paths -->

# Repository Documentation Information Architecture Design

## Goal

Resolve GitHub issue #6 by reorganizing the repository documentation into a
clear, maintainable information architecture while avoiding source-code layout
changes. The documentation should make it obvious which pages describe current
facts, which pages preserve design rationale, which pages are historical
implementation records, and which files are temporary compatibility stubs.

## Background

This repository is an FPGA quantum decoder exploration workspace, not a single
software package. It currently includes several active or parked modules:

- `surface_code_decoder/`: the Surface Code Python benchmark and decoder tooling.
- `verilog_test/`: the Verilog interface-level simulation harness.
- `qldpc_decoder/`: the QLDPC adaptive sparse decoder package.
- `GPU-decoder/`: a parked future module owned separately and not currently
  connected to the main validation flow.

Documentation is currently spread across the repository root, `docs/`,
`surface_code_decoder/docs/`, and `GPU-decoder/docs/`. Some files describe current behavior,
some are implementation handoff plans, and some are generated reports or
historical development artifacts. This makes it hard for new contributors and
agents to find the current source of truth.

## Non-Goals

- Do not move `surface_code_decoder/`, `verilog_test/`, `qldpc_decoder/`, or `GPU-decoder/`
  into a shared module directory as part of this issue.
- Document the current root-level module locations as the present repository
  state, not as a permanent source layout constraint. Future work may move
  independently developed modules under a shared module or project directory in
  a dedicated repository-layout migration.
- Do not change decoder logic, Verilog behavior, CI semantics, or generated
  vector formats.
- Do not migrate generated reports or runtime artifacts into long-term
  documentation navigation.
- Do not declare `GPU-decoder/` obsolete. It is parked for future integration.
- Do not rewrite historical plans into new implementation instructions.

## Proposed Documentation Structure

Long-term documentation should be consolidated under top-level `docs/`:

```text
docs/
  README.md
  architecture/
    overview.md
    surface-code-decoder.md
    python-verilog-interface.md
    qldpc-decoder.md
    gpu-decoder.md
  development/
    environment.md
    generated-artifacts.md
  verification/
    ci.md
    surface-code-verilog.md
    human-verification-test-plan.md
  specs/
    surface-code/
    interface/
    qldpc/
  plans/
    surface-code/
    interface/
    qldpc/
  archive/
    README.md
```

`docs/README.md` is the documentation navigation entry point. It explains which
section to use for current architecture, development setup, verification,
design specifications, implementation plans, and archived material.

## Information Layers

### Architecture Pages

Architecture pages are the current-fact layer. They summarize the present state
of the repository and link to deeper specs or plans when useful.

- `docs/architecture/overview.md` describes the repository as a multi-module
  FPGA quantum decoder exploration workspace.
- `docs/architecture/surface-code-decoder.md` describes the `surface_code_decoder/` Python
  benchmark and its relationship to Stim, PyMatching, latency modeling, and
  reporting.
- `docs/architecture/python-verilog-interface.md` describes the current
  Python-to-Verilog vector flow through `surface_code_decoder/`, `verilog_test/`, scripts,
  and CI. It must clearly state that the Verilog side is an interface contract
  model, not a hardware MWPM implementation.
- `docs/architecture/qldpc-decoder.md` describes `qldpc_decoder/` using a
  conservative status model based on current source files and tests.
- `docs/architecture/gpu-decoder.md` describes `GPU-decoder/` as a parked
  future module that is not part of the current validation path.

### Specs

Specs describe design rationale and intended contracts: what should exist, why
it exists, and where the system boundaries are. They remain distinct from
implementation plans.

Existing design specs should move into topic-specific directories:

- Surface Code specs: `docs/specs/surface-code/`
- Python-Verilog interface specs: `docs/specs/interface/`
- QLDPC specs: `docs/specs/qldpc/`

### Plans

Plans describe how a design was or should be implemented. They are useful
implementation records but are not the current source of truth once the source
tree has evolved.

Existing implementation plans should move into topic-specific directories:

- Surface Code plans: `docs/plans/surface-code/`
- Python-Verilog interface plans: `docs/plans/interface/`
- QLDPC plans: `docs/plans/qldpc/`

Moved plans that are historical or completed should preserve their content but
receive a short status note near the top:

```md
> Status: Historical implementation plan. The work described here may already
> be partially or fully implemented. Use the corresponding architecture page and
> current source tree as the source of truth.
```

### Development Pages

Development pages should be thin, stable references for common setup and output
conventions.

- `docs/development/environment.md` covers Python environment setup, editable
  installs, simulator dependencies, and common local command entry points.
- `docs/development/generated-artifacts.md` identifies generated artifacts such
  as `surface_code_decoder/benchmark-out/`, `verilog_test/vectors/current/`, pytest temp
  directories, simulator outputs, and cache directories. It should explain how
  to regenerate or inspect them without treating them as long-term docs.

### Verification Pages

Verification pages should separate automated and manual validation guidance.

- `docs/verification/ci.md` explains the GitHub Actions jobs and local
  equivalents.
- `docs/verification/surface-code-verilog.md` explains vector regeneration and
  Verilog interface self-check commands.
- `docs/verification/human-verification-test-plan.md` is the moved version of
  root `HUMAN_VERIFICATION_TEST_PLAN.md`.

### Archive

`docs/archive/README.md` should define archive rules. The archive is for
obsolete but still useful design drafts, plans, or verification notes. It is not
for generated reports, current source-of-truth docs, or runtime artifacts.

## Compatibility Stubs

Moved documents should leave short English stubs at their old paths during the
transition. Stubs should preserve the original title, identify the new path, and
state when the stub may be removed.

Example:

```md
# Python Decoder To Verilog Interface Simulation Design

This document moved to:

`../../../docs/specs/interface/2026-04-20-python-decoder-verilog-interface-design.md`

This compatibility stub can be removed after downstream references, open
branches, and agent guidance have been verified against the new path.
```

Compatibility stubs are expected for moved documents under:

- `docs/superpowers/`
- `surface_code_decoder/docs/superpowers/`
- `GPU-decoder/docs/superpowers/`
- root `HUMAN_VERIFICATION_TEST_PLAN.md`

## README And Agent Guidance

The root `README.md` should become a slim project entry point. It should
describe the repository purpose, list the core modules, explain that detailed
documentation lives under `docs/`, and avoid duplicating full verification
manuals.

`surface_code_decoder/README.md` should remain a subproject entry point and link to the
new documentation paths.

`qldpc_decoder/README.md` should retain its existing intent but update the
implementation status conservatively. The status should be based on current
source files and tests, using categories such as implemented, partial,
skeleton, and planned. It should not claim complete integration or performance
validation while `tests/test_integration.py` still contains skipped placeholder
tests.

`CLAUDE.md` should retain operational rules for agents while linking to stable
documentation pages where possible. `AGENTS.md` should remain only as a short
compatibility pointer to `CLAUDE.md`, which is the single source of truth for
agent guidance.

## Link Checker

Add a lightweight repository-local Markdown link checker:

```text
scripts/check_docs_links.py
```

The script should:

- scan Markdown files across the repository by default;
- skip generated and temporary paths such as benchmark outputs, pytest temp
  directories, simulator outputs, caches, and virtual environments;
- check local Markdown links for missing targets;
- ignore external links such as `http:`, `https:`, and `mailto:`;
- understand anchors by checking only the file target;
- report stale references to old documentation path prefixes in regular docs:
  - `docs/superpowers/`
  - `surface_code_decoder/docs/superpowers/`
  - `GPU-decoder/docs/superpowers/`
- allow compatibility stubs at moved old paths to mention their own historical
  locations or moved targets.

This checker should not be wired into CI as part of the first issue #6 change.
It should be runnable locally and available for future CI integration.

## Validation Strategy

The implementation should be validated with documentation-focused checks:

- Run the new Markdown link checker from the repository root.
- Use `rg` to confirm that old `superpowers` paths do not remain in regular
  documentation except compatibility stubs.
- Confirm `git status --short` and explain any remaining dirty files.
- Avoid running Python or Verilog behavioral test suites unless implementation
  work unexpectedly touches code, CI, scripts used by runtime flows, or command
  semantics.

## Acceptance Criteria

- `docs/README.md` clearly navigates architecture, development, verification,
  specs, plans, and archive sections.
- Surface Code, Python-Verilog interface, QLDPC, and parked GPU decoder module
  documentation have clear ownership and current-state summaries.
- Specs and plans remain distinct, with historical plans labeled appropriately.
- Moved documents leave English compatibility stubs at old paths with removal
  conditions.
- Root `README.md` is a concise project entry point rather than a duplicated
  verification manual.
- `CLAUDE.md` references the new documentation structure, and `AGENTS.md`
  points agents to `CLAUDE.md` as the single source of truth.
- `qldpc_decoder/README.md` gives a conservative status aligned with current
  source files and tests.
- Generated artifacts are described but not migrated into long-term docs.
- `scripts/check_docs_links.py` can detect missing local Markdown links and
  stale old documentation path references.
