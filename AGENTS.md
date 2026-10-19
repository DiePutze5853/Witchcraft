Here is the cleaned and properly formatted Markdown layout. All copy-paste artifacts (duplicated math expressions, fragmented LaTeX symbols, web citation tags, and UI labels like `code Bash`) have been resolved into clean Markdown and standard LaTeX math syntax:

---

# 1. Project Overview & Architecture

This repository contains an autonomous, closed-loop 3D mesh generation pipeline. It translates natural language prompts into production-grade 3D assets inside Blender via an end-to-end fluent Python Domain-Specific Language (DSL), driven by a fine-tuned multimodal model (Qwen/Qwen3.5-2B via Unsloth).

### Core Stack
* **Language & Runtime:** Python 3.11+, Blender 4.2+ LTS (embedded Python 3.11).
* **Modeling Engine:** Custom pure-3D fluent DSL $\rightarrow$ Blender `bpy` compiler.
* **Agent Stack:** Local persistent Blender daemon (FastAPI/WebSocket), AST security gateways.
* **Model & Training:** Qwen/Qwen3.5-2B using bf16 LoRA in Unsloth (~5GB training VRAM).
* **Validation & QA:** `trimesh`, `scikit-image` (SSIM render audits), `pytest`.

---

# 2. Repository Layout

```text
.
├── AGENTS.md                  # This file (Instructions for coding agents)
├── README.md                  # Human-facing project overview
├── ROADMAP.md                 # Master progress dashboard and gate policies
├── PHASE_1.md                 # Detailed spec: Core DSL, AST & Security
├── PHASE_2.md                 # Detailed spec: Blender Runtime & Finalizer
├── PHASE_3.md                 # Detailed spec: Pipeline & Orchestration
├── PHASE_4.md                 # Detailed spec: Validation & QA
├── PHASE_5.md                 # Detailed spec: Training & Benchmarks
├── pyproject.toml             # Project dependencies and tool configurations
├── core/
│   ├── dsl.py                 # Pure-3D fluent DSL nodes (Box, Cylinder, Hull, snap_to)
│   ├── ast_tree.py            # Intermediate Scene Graph representation
│   └── patch.py               # Critic Patch DSL parser & action recorder
├── security/
│   ├── validator.py           # Dual AST whitelist validators (Chunk & Patch)
│   └── sandbox.py             # Subprocess watchdog & execution sandboxing
├── runtime/
│   ├── blender_server.py      # Persistent FastAPI/WebSocket daemon inside Blender
│   ├── compiler.py            # DSL-to-bpy translation & mesh generation
│   ├── staging.py             # Transactional staging collection & rollback engine
│   └── finalizer.py           # Sequential boolean queue, adaptive epsilon, 1.0m normalization
├── pipeline/
│   ├── orchestrator.py        # 3-mode cyclic loop (Plan -> Code -> Finalize -> Critic)
│   ├── dag.py                 # Dependency graph & descendant invalidation engine
│   └── ledger.py              # State ledger & best-valid-state checkpoint recovery
├── training/
│   ├── flywheel.py            # Synthetic dataset generator (Teacher LLM -> Blender bench)
│   ├── train.py               # Unsloth Qwen3.5-2B bf16 LoRA training script
│   └── prompts.py             # System prompts for [MODE: PLAN], [MODE: CODE], [MODE: CRITIC]
├── validation/
│   ├── geometry.py            # Watertightness, degenerate faces, normal orientation audits
│   └── export_audit.py        # GLB export, re-import, and headless SSIM render verification
└── tests/
    ├── test_dsl.py            # Unit tests for DSL transforms & snapping
    ├── test_security.py       # Adversarial AST attacks and dunder access tests
    ├── test_runtime.py        # Transactional staging & boolean rollback tests
    └── test_integration.py    # End-to-end test (Plan -> Chunk -> Finalize -> GLB)
```

---

# 3. Development Setup & Commands

Always use the commands below. Do not guess build scripts or invent CLI tools.

### Environment Setup

```bash
# Create and activate virtual environment
python3.11 -m venv .venv
source .venv/bin/activate

# Install package in editable mode with dev & training dependencies
pip install -e ".[dev,training]"

# Verify Blender binary path (Blender 4.2+ LTS required)
export BLENDER_BIN="/Applications/Blender.app/Contents/MacOS/Blender" # macOS
# export BLENDER_BIN="/usr/bin/blender"                              # Linux
$BLENDER_BIN --version
```

### Running the Blender Runtime Daemon

```bash
# Start persistent background daemon on localhost:8765
$BLENDER_BIN --background --python runtime/blender_server.py -- --port 8765

# Start daemon with interactive GUI visible (for live visual debugging)
$BLENDER_BIN --python runtime/blender_server.py -- --port 8765
```

### Testing & Quality Checks

Run tests before committing any changes. All tests must be green.

```bash
# Run entire test suite
pytest -v

# Run AST security and adversarial validator tests
pytest tests/test_security.py -v

# Run transactional runtime tests (requires Blender daemon running)
pytest tests/test_runtime.py -v

# Run static typing and linting
mypy core security pipeline
ruff check .
ruff format --check .
```

### Benchmark & Training Commands

```bash
# Run synthetic dataset generation flywheel
python -m training.flywheel --num-samples 1000 --out-dir ./data/synthetic

# Train Qwen3.5-2B LoRA via Unsloth (requires GPU with >= 8GB VRAM)
python -m training.train --config training/configs/qwen35_bf16.yaml

# Evaluate checkpoint against the 200-prompt benchmark
python -m benchmarks.evaluate --checkpoint ./checkpoints/qwen35_lora_final
```

### 3.1 Autonomous Agent Execution Protocol

When executing work in this repository:
1. **Source of Truth:** Consult `ROADMAP.md` to identify the active Phase, and read `PHASE_<N>.md` for the exact granular task specifications.
2. **Sequential Progression:** Never begin tasks from `PHASE_<N+1>.md` until all acceptance criteria in `PHASE_<N>.md` are fully satisfied and verified.
3. **Task Check-off:** When completing a task item, edit `PHASE_<N>.md` to mark the task checkbox as completed (`- [x]`).
4. **Pre-Commit Verification:** Run the designated test commands and linting suite (`pytest`, `mypy`, `ruff`) before reporting task or phase completion.
5. **Dashboard Updates:** Update the status table in `ROADMAP.md` from `Pending` to `In Progress` or `Complete` as phases transition.

---

# 4. Architectural Invariants & Non-Negotiable Rules

Any AI agent modifying this repository must strictly preserve these core rules:

### 4.1 Strict Domain Separation (`bpy` Isolation)
* **Rule:** Never import `bpy`, `bmesh`, or `mathutils` inside `core/`, `security/`, `pipeline/`, or `training/`.
* `core/dsl.py` contains only declarative Python dataclasses and method-chaining builders that emit an Abstract Syntax Tree (AST).
* Only files inside `runtime/` are allowed to import Blender’s `bpy` API. All communication from the orchestrator to Blender passes through `runtime/blender_server.py`.

### 4.2 Pure 3D Volume Modeling (No 2D, No Vertex Soup)
* There are no 2D sketch arrays, no 2D profile revolves, and no face/vertex index lookups in the DSL.
* The primitive vocabulary is strictly limited to: `Box`, `Cylinder`, `Cone`, `Sphere`, `Torus`, and `Hull`.
* Irregular organic transitions are accomplished strictly via `Hull([prim1, prim2, ...])` and procedural surface `.displace()`.

### 4.3 Units & Scale Normalization
* **Canonical Working Units (CWU):** The Coder models in local relative units ($1\text{ CWU} \equiv 1\text{m}$ pre-normalization, with dominant root extents $\approx 0.5 - 1.0\text{ CWU}$).
* **Assembly Normalization:** The Planner never outputs metric values. Assembly normalization is an automated post-process: during `/scene/finalize`, after all booleans execute, the entire composite assembly is centered and uniformly scaled to fit exactly within a 1.0m bounding cube:

$$\text{Scale Factor} = \frac{1.0}{\max(\Delta X, \Delta Y, \Delta Z)}$$

### 4.4 Transform Evaluation Order
Chained transformations on `Node3D` must be evaluated deterministically in this exact mathematical order:

$$\text{Output Transform} = \text{Move}(\Delta x, \Delta y, \Delta z) \times \text{SnapTo}(\text{Target}, \text{Face}) \times \text{Rotate}(\theta_x, \theta_y, \theta_z) \times \text{Scale}(s_x, s_y, s_z) \times \text{BaseMesh}$$

*(Calling `.snap_to()` defines the base translation; calling `.move()` applies local offsets on top of the snapped position).*

### 4.5 Dual Whitelist AST Security (Never Denylists)
* Never use denylists (`ast.Exec` does not exist in Python 3.11+ and will crash).
* Use the two distinct whitelist visitors in `security/validator.py`:
  * **`ChunkASTValidator`:** Validates `build_<id>(scene)` definitions. Blocks all private/dunder access (`attr.startswith('_')`), allows only mathematical operations, dictionary lookups on `scene.get()`, registered DSL primitives, and method chaining on `Node3D`.
  * **`PatchASTValidator`:** Validates standalone Critic patch statements (`ast.Expr`). Allows only root calls to `approve(...)` and `revise(...)`, chained with `.scale()`, `.move()`, `.adjust()`, and `.note()`.
* **Execution Limit:** 5-second hard timeout per chunk execution via subprocess watchdog.

### 4.6 Transactional Staging & Rollback
* Chunks execute into an isolated Blender collection (`__staging__`).
* Chunks are inspected by the geometric acceptance suite before being committed to `__active_scene__`.
* If a chunk fails validation, `__staging__` is purged immediately. The active scene remains uncorrupted.

### 4.7 Transactional Boolean Finalization & Blame Attribution
* During `/scene/finalize`, boolean cuts execute sequentially from a dependency queue.
* A mesh snapshot is created before each boolean operation.
* If a cut fails, the target geometry is restored from snapshot, the failure is mapped directly to the responsible cutter chunk, and the adaptive epsilon offset is applied:

$$\epsilon = \operatorname{clamp}\left(\min(\operatorname{dim}_{\min}(A), \operatorname{dim}_{\min}(B)) \times 0.005,\, 0.0005,\, 0.005\right)$$

* Never leave a partially modified scene upon finalization error.

### 4.8 DAG-Aware Patch Propagation
* When the Critic revises chunk $C$, the orchestrator must invalidate $C$ and all topological descendants of $C$ in the Planner assembly graph.
* Descendant chunks must be automatically re-executed via their deterministic `build_*` functions so their `.snap_to()` anchors attach to the updated boundary coordinates.

### 4.9 Best-Valid-State Recovery (Fail-Closed)
* Never average contradictory patch parameters.
* Scale patches are strictly multiplicative:
  $$x_{\text{new}} = x_{\text{old}} \times s_x$$
* Translation patches are strictly additive:
  $$z_{\text{new}} = z_{\text{old}} + dz$$
* If anti-oscillation triggers or the 3-loop revision budget expires, the pipeline rolls back to the highest-scoring valid checkpoint in `SceneLedger`.
* **Fail-Closed:** If no state meets all mandatory geometric checks, raise `PipelineFailureError`. Never export an unverified mesh.

---

# 5. Geometric Acceptance & Export Criteria

All chunks and final assemblies must pass these strict programmatic tests in `validation/`:

* **2-Manifold Watertightness:** `trimesh.is_watertight == True`. Every edge must connect to exactly two faces.
* **Degenerate Faces:** Zero faces with surface area $< 10^{-6}\text{ CWU}^2$.
* **Solid-Relative Outward Normals:** Face normals must point outward relative to solid material (exterior faces point out into world space; interior cavity faces point inward into the empty void).
* **Self-Intersections:** Zero self-intersecting coplanar faces (`trimesh.intersections`).
* **Roundtrip Render Similarity:** The exported `.glb` must be re-imported into a clean verification scene and rendered headlessly. Structural Similarity Index against the pre-export render must satisfy:

$$\text{SSIM}(\text{Render}_{\text{Blender}}, \text{Render}_{\text{GLB}}) \ge 0.92$$

---

# 6. Model Training & Prompting Contracts

### 6.1 Unsloth Training Configuration (Qwen/Qwen3.5-2B)
* **Precision:** bf16 LoRA (`bfloat16 = True`, `load_in_4bit = False`). Do not use 4-bit QLoRA on Qwen3.5 due to quantization differences.
* **LoRA Target Modules:** `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj`.
* **Hyperparameters:** `r = 16`, `lora_alpha = 32`, `lora_dropout = 0.05`, `learning_rate = 2e-4`, `optimizer = "adamw_8bit"`.
* **Loss Masking:** Mask out all prompt, user, system, and `<image>` embedding tokens (`label = -100`). Compute loss strictly on completion tokens (`<|im_start|>assistant` to `<|im_end|>`).

### 6.2 Mode Output Templates

#### `[MODE: PLAN]` Output Format
```markdown
[MODE: PLAN]
TARGET: <Object Name>
ASSEMBLY_ORDER:
1. <root_component> (Root)
2. <child_component> (Depends on: <parent>)
...
COMPONENTS:
- <component_name>:
    primitive: <Box | Cylinder | Cone | Sphere | Torus | Hull>
    scale_class: <PRIMARY | SECONDARY | DETAIL>
    aspect_ratio: [x, y, z]
    geometry_notes: <Description>
    anchors: <RESTS_ON_CENTER | ATTACHED_TO | CARVED_INTO> -> <parent>.<FACE>
...
MATERIAL:
- <MatName> (Color: (r, g, b), Metallic: m, Roughness: r)
```

#### `[MODE: CODE]` Output Format
```python
# [MODE: CODE]
# Component: <component_name>
def build_<component_name>(scene: SceneContext) -> Node3D:
    # 1. Fetch parent reference via scene.get() if needed
    parent = scene.get("<parent_name>")
    # 2. Instantiate primitive, snap, transform, name
    return (
        Cylinder(r=0.08, h=0.25)
        .snap_to(parent, on="TOP")
        .name("<component_name>")
    )
```

#### `[MODE: CRITIC]` Output Format
```python
# Pass
approve(score=9.2, note="Proportions accurate, manifold verified")

# Revision
revise("<component_name>").adjust(r=0.06, h=0.32).note("<actionable reason>")
```

---

# 7. Prohibited Patterns (NEVER DO)

* **NEVER** edit files under `runtime/` to accept raw arbitrary Python execution. All code paths must pass through `ChunkASTValidator` or `PatchASTValidator`.
* **NEVER** use denylists or string regex matching for security validation.
* **NEVER** introduce 2D point arrays (`Extrude2D`, `Revolve2D`) or polygon vertex coordinates.
* **NEVER** emit hardcoded metric coordinates ($z = 0.75$) in `[MODE: PLAN]`.
* **NEVER** use naive parameter averaging when anti-oscillation triggers. Always roll back to the best verified state in `SceneLedger`.
* **NEVER** bypass transactional staging by adding objects directly to `bpy.context.scene.collection`.
* **NEVER** force-export a broken or non-manifold mesh when loop iterations expire. Fail closed and report the error.