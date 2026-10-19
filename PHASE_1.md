# Phase 1 — Sprint 1: Core DSL Engine, Scene Representation & AST Security Gateway

**Objective:** Implement the pure-3D Fluent DSL, intermediate scene graph structures, the Critic Patch language, and the dual AST whitelist security sandbox. No Blender (`bpy`) dependencies are introduced in this sprint; all modules must remain decoupled, deterministic, and statically verifiable.

---

### Sprint 1 Task Breakdown

#### Task 1.1: Project Initialization & Tooling Baseline
- [ ] Create repository skeleton matching Section 2 of `AGENTS.md`:
  - Directories: `core/`, `security/`, `runtime/`, `pipeline/`, `training/`, `validation/`, `tests/`.
  - Place `__init__.py` files across all packages.
- [ ] Implement `pyproject.toml`:
  - Python version requirement: `>=3.11`.
  - Base dependencies: `pydantic>=2.0`, `numpy`, `typing-extensions`.
  - Optional dev dependencies: `pytest`, `mypy`, `ruff`.
  - Configure tool sections: `ruff` (formatting and linting rules), `mypy` (strict type checking across `core/` and `security/`).
- [ ] **Acceptance Criteria:** `ruff check .`, `ruff format --check .`, and `mypy core security` execute cleanly.

---

#### Task 1.2: Fluent Pure-3D DSL Implementation (`core/dsl.py`)
- [ ] Implement primitive volume dataclasses inheriting from a base `Node3D`:
  - `Box(x: float, y: float, z: float)` (require $x, y, z > 0$).
  - `Cylinder(r: float, h: float, r_top: float | None = None)` ($r, h > 0$; optional frustum parameter).
  - `Cone(r_base: float, r_top: float = 0.0, h: float = 1.0)`.
  - `Sphere(r: float)` ($r > 0$).
  - `Torus(major_r: float, minor_r: float)` ($major\_r > minor\_r > 0$).
  - `Hull(primitives: list[Node3D])` (enforce minimum of 2 primitives; reject empty or 1-element lists).
- [ ] Implement method chaining on `Node3D`:
  - `.at(x: float = 0, y: float = 0, z: float = 0)` (absolute translation record).
  - `.move(dx: float = 0, dy: float = 0, dz: float = 0)` (relative delta translation).
  - `.rotate(x: float = 0, y: float = 0, z: float = 0)` (Euler angles in degrees).
  - `.scale(x: float = 1.0, y: float = 1.0, z: float = 1.0)` (scale factors $> 0$).
  - `.snap_to(target: Node3D, on: Literal["TOP", "BOTTOM", "FRONT", "BACK", "RIGHT", "LEFT"])`.
  - `.union(mesh_b: Node3D)`, `.difference(mesh_b: Node3D)`, `.intersect(mesh_b: Node3D)`.
  - `.as_difference_for(target_names: list[str])`.
  - Modifiers: `.displace(strength: float, roughness: float = 1.0)`, `.bevel(radius: float, segments: int = 2)`, `.mirror(axis: Literal["X", "Y", "Z"])`.
  - Attributes: `.material(name: str, color: tuple[float, float, float], roughness: float = 0.5, alpha: float = 1.0, metallic: float = 0.0)`, `.name(chunk_id: str)`.
- [ ] Ensure transform metadata strictly encodes the mandatory evaluation order:
  $$\text{Output Transform} = \text{Move}(\Delta x, \Delta y, \Delta z) \times \text{SnapTo}(\text{Target}, \text{Face}) \times \text{Rotate}(\theta_x, \theta_y, \theta_z) \times \text{Scale}(s_x, s_y, s_z) \times \text{BaseMesh}$$
- [ ] **Invariants Check:** Ensure zero references to 2D sketch arrays, raw vertex lists, or dynamic mesh face indices. Under no circumstances import `bpy`, `bmesh`, or `mathutils`.
- [ ] **Acceptance Criteria:** Chained node definitions execute in memory and produce deterministic, inspectable transformation descriptors.

---

#### Task 1.3: Intermediate Scene Representation & Scene Context (`core/ast_tree.py`)
- [ ] Implement `SceneContext`:
  - Dict-like interface providing `scene.get(name: str) -> Node3D`.
  - Internal registry storing staged/committed nodes.
  - Bounding box abstraction (`BoundingBox3D(min_point, max_point)`) with centroid calculation and 6 face anchors (`TOP`, `BOTTOM`, `FRONT`, `BACK`, `RIGHT`, `LEFT`) to enable offline resolution of `.snap_to()` rules.
- [ ] Implement `SceneGraph`:
  - Tracks chunk execution state (`REGISTERED`, `STAGED`, `COMMITTED`).
  - Serializes node parameters to intermediate dictionaries for cross-process IPC.
- [ ] **Acceptance Criteria:** `SceneContext` allows lookup of committed nodes, and snapping against mock bounding boxes resolves correct 3D translations.

---

#### Task 1.4: Critic Patch DSL Parser & Action Recorder (`core/patch.py`)
- [ ] Implement fluent patch builders:
  - `approve(score: float, note: str)`: Returns an `ApprovalRecord`.
  - `revise(target_chunk: str)`: Returns a `RevisionPatch` builder supporting:
    - `.scale(x: float = 1.0, y: float = 1.0, z: float = 1.0)` (multiplicative updates: $x_{\text{new}} = x_{\text{old}} \times s_x$).
    - `.move(dx: float = 0.0, dy: float = 0.0, dz: float = 0.0)` (additive updates: $z_{\text{new}} = z_{\text{old}} + dz$).
    - `.adjust(**kwargs)` (named primitive parameter overrides like `r=0.06, h=0.32`).
    - `.note(reason: str)`.
- [ ] Add serialization methods to convert patch objects into JSON payloads for the orchestrator.
- [ ] **Acceptance Criteria:** `revise("neck").adjust(r=0.06, h=0.32).note("Too thick")` successfully parses into a typed, structured action payload.

---

#### Task 1.5: Dual AST Whitelist Security Validators (`security/validator.py`)
- [ ] Implement `COMMON_ALLOWED_NODES` whitelist containing safe basic Python syntax:
  - `ast.Module`, `ast.Constant`, `ast.Name`, `ast.Call`, `ast.keyword`, `ast.Attribute`, `ast.BinOp`, `ast.UnaryOp`, `ast.Add`, `ast.Sub`, `ast.Mult`, `ast.Div`, `ast.USub`, `ast.UAdd`, `ast.Load`.
- [ ] Implement `ChunkASTValidator(ast.NodeVisitor)`:
  - Allowed AST nodes: `COMMON_ALLOWED_NODES | {ast.FunctionDef, ast.arguments, ast.arg, ast.Return, ast.Assign, ast.List, ast.Tuple, ast.Store}`.
  - Allowed Primitives: `{"Box", "Cylinder", "Cone", "Sphere", "Torus", "Hull"}`.
  - Recursion depth limiter: enforce `max_depth = 20` to block AST stack-overflow exploits.
  - Block all dunder/private attribute access: `node.attr.startswith("_")`.
  - Allowed call targets: Registered DSL primitives, `scene.get("...")`, and exposed `Node3D` methods.
  - Block all `ast.Import`, `ast.ImportFrom`, global function calls (`eval`, `exec`, `open`, `__import__`).
- [ ] Implement `PatchASTValidator(ast.NodeVisitor)`:
  - Allowed AST nodes: `COMMON_ALLOWED_NODES | {ast.Expr}`.
  - Allowed root calls: `{"approve", "revise"}`.
  - Allowed patch methods: `{"scale", "move", "adjust", "note"}`.
  - Block all imports, statements, assignments, and nested function calls.
- [ ] **Acceptance Criteria:** Any AST containing disallowed nodes or private accesses raises a custom `SecurityError` with zero bypasses.

---

#### Task 1.6: Subprocess Execution Sandbox & Watchdog (`security/sandbox.py`)
- [ ] Implement `ExecutionSandbox`:
  - Wraps code execution inside an isolated subprocess via Python’s `multiprocessing` or `subprocess`.
  - Enforces a **5.0-second hard watchdog timeout**.
  - Captures `stdout`, `stderr`, and exceptions.
  - Sanitizes execution scope: only provide `SceneContext` and exposed DSL symbols in the execution namespace.
- [ ] **Acceptance Criteria:** An intentional infinite loop (`while True: pass`) terminates and raises an `ExecutionTimeoutError` at exactly 5 seconds.

---

#### Task 1.7: Test Harness & Verification Suite (`tests/`)
- [ ] Create `tests/test_dsl.py`:
  - Unit test each primitive instantiation and validation rules (positive bounds checks).
  - Verify deterministic transform chaining math (Move $\times$ Snap $\times$ Rotate $\times$ Scale).
  - Test `.snap_to()` offset calculations across all 6 directional faces (`TOP`, `BOTTOM`, etc.).
  - Verify boolean cut tagging via `.as_difference_for()`.
- [ ] Create `tests/test_security.py`:
  - Test adversarial attack vectors: `__subclasses__()`, `import os`, `eval()`, `open()`, builtins access, global variable modification.
  - Test deep nesting AST exploits (`max_depth > 20`).
  - Test patch validator against arbitrary execution injection inside `approve()` / `revise()`.
  - Test sandbox 5-second watchdog termination.
- [ ] **Acceptance Criteria:** `pytest tests/test_dsl.py tests/test_security.py -v` passes with 100% green status and 0 static typing warnings.

---