# Autonomous 3D Generation Pipeline — Master Roadmap

This document tracks high-level progress and milestones across all engineering phases. Detailed, task-by-task specifications for autonomous agents are maintained in modular phase files: `PHASE_1.md` through `PHASE_5.md`.

---

## Phase Status Dashboard

| Phase | Title | Spec File | Status | Gate Requirement |
| :--- | :--- | :--- | :--- | :--- |
| **Phase 1** | Core DSL, AST Tree & Security Validators | `PHASE_1.md` | ⏳ Pending | `pytest tests/test_dsl.py tests/test_security.py` 100% green |
| **Phase 2** | Blender Runtime, Staging Buffer & Finalizer | `PHASE_2.md` | ⏳ Pending | `pytest tests/test_runtime.py` green with live Blender daemon |
| **Phase 3** | Pipeline Orchestrator, DAG Engine & Ledger | `PHASE_3.md` | ⏳ Pending | `pytest tests/test_pipeline.py` green, mock loop passing |
| **Phase 4** | Geometric Acceptance Suite & Export Audit | `PHASE_4.md` | ⏳ Pending | `pytest tests/test_integration.py` passing, SSIM >= 0.92 |
| **Phase 5** | Data Flywheel, Fine-Tuning & Evaluation | `PHASE_5.md` | ⏳ Pending | Evaluation benchmark script operational with 95% Wilson CIs |

---

## Phase Summaries & Key Deliverables

### Phase 1: Core DSL Engine, Intermediate AST & Security Gateway
* **Focus:** Declarative 3D primitives (`Box`, `Cylinder`, `Cone`, `Sphere`, `Torus`, `Hull`), chaining methods, `SceneContext`, Critic Patch builders, dual AST whitelist security visitors, and a 5-second execution watchdog.
* **Invariant:** Pure Python only; zero imports of `bpy`, `bmesh`, or `mathutils`.
* **Details:** See `PHASE_1.md`.

### Phase 2: Blender Runtime, Staging Buffer & Transactional Finalizer
* **Focus:** Headless/GUI background daemon on port 8765, DSL-to-`bpy` compiler, isolated transactional collections (`__staging__` vs `__active_scene__`), sequential boolean cuts with isolated rollback snapshots, adaptive epsilon offset solver, and 1.0m normalization.
* **Invariant:** All Blender operations remain inside `runtime/`.
* **Details:** See `PHASE_2.md`.

### Phase 3: Pipeline Orchestrator, DAG Engine & State Ledger
* **Focus:** Directed Acyclic Graph engine for component dependencies, topological descendant invalidation and re-anchoring, append-only `SceneLedger`, anti-oscillation parameter monitoring, and fail-closed resolution.
* **Invariant:** Communication with Blender occurs exclusively via IPC.
* **Details:** See `PHASE_3.md`.

### Phase 4: Geometric Acceptance Suite & Export Verification
* **Focus:** 2-manifold watertight verification (`trimesh.is_watertight`), zero degenerate faces ($< 10^{-6}\text{ CWU}^2$), outward solid-relative normal audits, glTF/GLB PBR material checks, and headless roundtrip render SSIM verification ($\ge 0.92$).
* **Invariant:** Strict fail-closed policy: never export an unverified mesh.
* **Details:** See `PHASE_4.md`.

### Phase 5: Synthetic Data Flywheel, Fine-Tuning & Benchmark
* **Focus:** Synthetic prompt/script flywheel with triplet mining for visual critique, Unsloth `Qwen/Qwen3.5-2B` bf16 LoRA fine-tuning with strict assistant loss masking, and a 200-prompt quantitative benchmark harness.
* **Invariant:** `load_in_4bit = False`, `bfloat16 = True`, loss computed only on assistant tokens.
* **Details:** See `PHASE_5.md`.

---

## Milestone Gate Policy

Before marking any Phase as **Complete** in the dashboard above:
1. All task checkboxes in `PHASE_<N>.md` must be checked (`[x]`).
2. All corresponding unit, security, and integration tests must pass.
3. Static typing and linting must pass cleanly (`ruff check .`, `ruff format --check .`, `mypy <target_packages>`).
4. No forbidden architectural invariants may be violated.