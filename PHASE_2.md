# Phase 2 — Sprint 2: Blender Runtime, Staging Buffer & Transactional Finalizer

**Objective:** Implement the isolated execution environment running inside Blender 4.2+ LTS. This includes the persistent FastAPI/WebSocket IPC daemon, the DSL-to-`bpy` compiler, the transactional staging buffer (`__staging__` vs `__active_scene__`), sequential boolean cuts with isolated rollback snapshots, blame attribution, adaptive epsilon recovery, and automated 1.0-meter assembly normalization.

---

### Sprint 2 Task Breakdown

#### Task 2.1: Blender Daemon & IPC Server (`runtime/blender_server.py`)
- [ ] Implement headless/GUI server executable directly inside Blender’s embedded Python 3.11:
  - CLI bootstrap: support both headless (`--background --python ... -- --port <port>`) and windowed debugging modes.
  - Set up lightweight ASGI server (FastAPI + Uvicorn) or WebSocket handler running in a non-blocking background thread via Blender’s modal operator / timer API (`bpy.app.timers`).
- [ ] Define HTTP / WebSocket API contracts:
  - `POST /chunk/execute`: Receives raw chunk Python code string + target chunk ID.
  - `POST /scene/finalize`: Triggers sequential boolean resolution, blame checks, and normalization.
  - `GET /scene/state`: Returns active object manifest, bounding boxes, and transform metadata.
  - `POST /scene/reset`: Flushes scene datablocks for a clean test/run slate.
- [ ] Integrate AST gateway:
  - Pass all incoming chunk code through `ChunkASTValidator` before execution.
  - Reject unauthorized payloads with HTTP 403 / Security error before evaluating in Blender.
- [ ] **Acceptance Criteria:** Blender launches headlessly, starts the server on `--port 8765`, passes the security gateway check, and responds to ping/status requests.

---

#### Task 2.2: DSL-to-`bpy` Geometric Compiler (`runtime/compiler.py`)
- [ ] Implement primitive builders using `bmesh` / `bpy.ops.mesh`:
  - `Box(x, y, z)`: Centered mesh primitive with exact edge dimensions.
  - `Cylinder(r, h, r_top)`: Cylinder or truncated cone using `bmesh.ops.create_cone` (matching base and top radii).
  - `Cone(r_base, r_top, h)`: Circular cone / truncated frustum.
  - `Sphere(r)`: Centered UV sphere with consistent latitude/longitude resolution.
  - `Torus(major_r, minor_r)`: Torus aligned to the XY plane.
  - `Hull(primitives)`: Merges vertices of input primitives into a single bmesh and applies `bmesh.ops.convex_hull` to produce a single watertight manifold.
- [ ] Implement deterministic transformation evaluation order:
  $$\text{Output Transform} = \text{Move}(\Delta x, \Delta y, \Delta z) \times \text{SnapTo}(\text{Target}, \text{Face}) \times \text{Rotate}(\theta_x, \theta_y, \theta_z) \times \text{Scale}(s_x, s_y, s_z) \times \text{BaseMesh}$$
- [ ] Implement snapping math:
  - Calculate exact source and target bounding boxes along world axes.
  - Offset source mesh so its bounding plane aligns flush against target face (`TOP` aligns source bottom to target top along $+Z$; `BOTTOM` aligns source top to target bottom along $-Z$, etc.) with centroids matched on the orthogonal axes.
- [ ] Implement modifiers and material assignments:
  - `.bevel()`: Non-destructive Bevel modifier or `bmesh.ops.bevel` with width clamping.
  - `.displace()`: Procedural surface noise using Blender displacement modifier with procedural Clouds texture.
  - `.material()`: Creates or fetches Principled BSDF node tree, binding base color, roughness, metallic, and transmission/alpha properties.
- [ ] **Acceptance Criteria:** Every DSL primitive compiles cleanly inside Blender into a valid `bpy.types.Object` with exact dimensions and proper transforms.

---

#### Task 2.3: Transactional Staging & Rollback Engine (`runtime/staging.py`)
- [ ] Create isolated Blender collections:
  - `__staging__`: Holds newly instantiated chunk geometry during execution and preliminary evaluation.
  - `__active_scene__`: Holds committed chunks.
- [ ] Implement transactional lifecycle:
  1. **Stage:** Execute chunk inside `__staging__` collection.
  2. **Audit:** Evaluate intermediate geometry (verify non-empty mesh, valid faces, bounded dimensions).
  3. **Commit:** Link object to `__active_scene__`, unlink from `__staging__`, update registered bounding boxes in `SceneContext`.
  4. **Rollback:** On any compilation, syntax, or geometric failure, unlink and delete all datablocks created in `__staging__` (meshes, materials, objects) and leave `__active_scene__` pristine.
- [ ] **Acceptance Criteria:** A faulty chunk script that raises an error leaves zero residual datablocks or orphaned objects in Blender memory.

---

#### Task 2.4: Sequential Boolean Queue & Adaptive Epsilon Solver (`runtime/finalizer.py`)
- [ ] Implement dependency-sorted boolean queue:
  - Discover chunks tagged with `.as_difference_for([target_ids])` or `.union()` / `.difference()` / `.intersect()`.
  - Process operations sequentially one at a time.
- [ ] Implement per-operation snapshot & blame attribution:
  - Clone target mesh datablock (`target_obj.data.copy()`) prior to applying the cut.
  - Apply boolean modifier (using `EXACT` or `FLOAT` solver).
  - Inspect intermediate topology. If boolean fails (results in non-manifold edges or mesh degradation):
    - Revert target object data from snapshot.
    - Attribute fault directly to the cutter chunk ID.
    - Trigger automated recovery pass via adaptive epsilon offset:
      $$\epsilon = \operatorname{clamp}\left(\min(\operatorname{dim}_{\min}(A), \operatorname{dim}_{\min}(B)) \times 0.005,\, 0.0005,\, 0.005\right)$$
    - Expand cutter along cut face normal by $\pm\epsilon$ and re-attempt cut.
    - If recovery fails after epsilon expansion, discard operation, restore snapshot, and return targeted error log identifying the failing cutter chunk.
- [ ] **Acceptance Criteria:** Intersecting/coplanar boolean cuts either succeed via adaptive epsilon expansion or roll back cleanly to pre-cut snapshots while emitting blame logs.

---

#### Task 2.5: Assembly Centering & 1.0m Extent Normalization (`runtime/finalizer.py`)
- [ ] Implement assembly normalization:
  - Calculate global composite bounding box across all committed objects in `__active_scene__`:
    $$\Delta X = X_{\max} - X_{\min}, \quad \Delta Y = Y_{\max} - Y_{\min}, \quad \Delta Z = Z_{\max} - Z_{\min}$$
  - Calculate global scale factor:
    $$\text{Scale Factor} = \frac{1.0}{\max(\Delta X, \Delta Y, \Delta Z)}$$
  - Uniformly scale all objects around the global center.
  - Translate the composite assembly centroid to origin $(0, 0, 0)$ or base plane $Z_{\min} = 0.0$ per spec.
  - Apply transforms (`bpy.ops.object.transform_apply(location=True, rotation=True, scale=True)`).
- [ ] **Acceptance Criteria:** Finalized multi-part assembly fits inside a $1.0\text{m}$ bounding cube with maximum dimension equal to $1.0\text{m}$.

---

#### Task 2.6: Runtime Integration & Transactional Verification Suite (`tests/test_runtime.py`)
- [ ] Create live runtime tests (executed against background Blender instance):
  - Test staging isolation: verify that invalid code inside `__staging__` leaves `__active_scene__` unchanged.
  - Test compiler transforms: verify snapped placement of cylinder onto box matches expected world coordinates.
  - Test boolean rollback and blame: simulate coplanar failure, verify snapshot recovery and error mapping.
  - Test finalization normalization: verify max dimension scales accurately to $1.0\text{m}$.
- [ ] **Acceptance Criteria:** `pytest tests/test_runtime.py -v` passes with all tests green against a running Blender daemon.

---