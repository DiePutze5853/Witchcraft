# Phase 3 — Sprint 3: Pipeline Orchestrator, DAG Engine & State Ledger

**Objective:** Implement the closed-loop multi-mode agent orchestration system (`pipeline/`). This module coordinates the execution across `[MODE: PLAN]`, `[MODE: CODE]`, and `[MODE: CRITIC]`, manages the component dependency DAG, invalidates and re-anchors downstream components upon revision, and maintains a versioned state ledger with anti-oscillation detection and fail-closed rollback.

*Strict Architectural Invariant:* Under no circumstances may files in `pipeline/` import `bpy`, `bmesh`, or `mathutils`. All interactions with Blender must occur via IPC client calls to `runtime/blender_server.py`.

---

### Sprint 3 Task Breakdown

#### Task 3.1: Assembly DAG Engine & Descendant Invalidation (`pipeline/dag.py`)
- [ ] Implement `AssemblyDAG`:
  - Parses `ASSEMBLY_ORDER` and component anchor dependencies from `[MODE: PLAN]` payloads.
  - Represents components as nodes with directed edges indicating anchor dependencies (e.g., `neck` $\rightarrow$ `flask_body`).
  - Implements cycle detection (raises `CyclicDependencyError` if plan contains circular references).
  - Provides topological sort for deterministic sequential generation order.
- [ ] Implement DAG Invalidation Engine:
  - `get_descendants(chunk_id: str) -> list[str]`: Traverses the graph from a revised target chunk and returns all downstream children in topological order.
  - `invalidate(chunk_id: str)`: Marks the target chunk and all transitive descendants as dirty/stale, scheduling them for deterministic re-execution.
- [ ] **Acceptance Criteria:** Given a graph `flask_body -> neck -> rim`, revising `neck` invalidates `neck` and `rim`, but leaves `flask_body` intact.

---

#### Task 3.2: Versioned State Ledger & Anti-Oscillation Monitor (`pipeline/ledger.py`)
- [ ] Implement `SceneLedger`:
  - Maintains an append-only timeline of scene checkpoints: `Checkpoint(iteration: int, state_id: str, scene_metadata: dict, critic_score: float, is_valid: bool, code_manifest: dict[str, str])`.
  - Links checkpoints to serialized mesh datablocks / snapshot IDs maintained in the runtime.
  - `get_best_valid_checkpoint() -> Checkpoint | None`: Queries the ledger for the state with the highest aesthetic score that passed all geometric validation tests.
- [ ] Implement Anti-Oscillation Detection:
  - Tracks parameter modification deltas across consecutive iterations for each chunk (e.g., `s_z` history, translation history).
  - Detects contradictory oscillation: flagging if opposite adjustments on the same parameter occur consecutively (e.g., $s_z = 1.25$ followed immediately by $s_z = 0.8$, or $dz = +0.10$ followed by $dz = -0.09$).
  - When oscillation is detected: aborts current revision branch and triggers best-valid-state rollback.
- [ ] Implement Fail-Closed Resolution:
  - If critique budget (3 cycles) expires or oscillation triggers: roll back to `get_best_valid_checkpoint()`.
  - If no checkpoint in the ledger has passed 100% of the geometric acceptance tests, raise `PipelineFailureError`. Never force an export of broken geometry.
- [ ] **Acceptance Criteria:** Repeated contradictory patch commands trip the oscillation detector, abort further cycles, and roll back cleanly to the best verified checkpoint.

---

#### Task 3.3: Blender Runtime IPC Client (`pipeline/client.py`)
- [ ] Implement `BlenderClient`:
  - Lightweight HTTP/WebSocket client communicating with `runtime/blender_server.py`.
  - Endpoints wrapped:
    - `stage_chunk(chunk_id: str, code_str: str) -> dict`: Dispatches chunk execution.
    - `finalize_scene() -> dict`: Dispatches sequential booleans and normalization.
    - `get_scene_state() -> dict`: Retrieves bounding boxes and topology statistics.
    - `render_multiview() -> bytes`: Requests the $1536 \times 512$ diagnostic render strip (Front ortho, Side ortho, Iso perspective).
    - `reset_scene()`: Clears the workspace.
  - Handles network timeouts and connection retries.
- [ ] **Acceptance Criteria:** `BlenderClient` successfully stages chunks, triggers boolean finalization, and receives serialized responses from a background Blender process.

---

#### Task 3.4: Multi-Mode Cyclic Orchestrator (`pipeline/orchestrator.py`)
- [ ] Implement `Orchestrator`:
  - Drives the full generation lifecycle:
    1. **Plan Ingestion:** Ingests `[MODE: PLAN]` output, initializes `AssemblyDAG`.
    2. **Sequential Chunk Loop:** Iterates through topological order, invoking chunk builders one component at a time.
    3. **Fast-Fail Gate:** For each chunk in `[MODE: CODE]`:
       - Run AST security validator (`ChunkASTValidator`).
       - If AST validation fails or execution raises an exception: feed error traceback back into the code generator for up to **3 automated retries**.
       - Abort entire pipeline if retries are exhausted without a clean execution.
    4. **Transactional Finalization:** Calls `/scene/finalize` via `BlenderClient` to execute sequential boolean operations, adaptive epsilon fixes, and 1.0m normalization.
    5. **Critic Evaluation Loop:**
       - Obtain multi-view diagnostic image strip and geometric metadata.
       - Query Critic (`[MODE: CRITIC]`).
       - Validate critic statement using `PatchASTValidator`.
       - On `approve(score, note)`: Record approval in `SceneLedger` and terminate loop with success.
       - On `revise(...)`: Dispatch patch to Task 3.5.
- [ ] **Acceptance Criteria:** A full assembly script builds end-to-end, stages chunks one-by-one, handles syntax retries, and stops upon Critic approval.

---

#### Task 3.5: Downstream Re-anchoring & Patch Propagation Loop
- [ ] Implement patch propagation pipeline:
  - Parses validated `RevisionPatch` (`.scale()`, `.move()`, `.adjust()`, `.note()`).
  - Applies parameter adjustments to the targeted chunk's code or parameter registry:
    - Multiplicative scaling: $x_{\text{new}} = x_{\text{old}} \times s_x$.
    - Additive translation: $z_{\text{new}} = z_{\text{old}} + dz$.
    - Parameter overrides: update constructor arguments (e.g., $r$, $h$).
  - Invokes `AssemblyDAG.invalidate(target_chunk)`.
  - Sequentially re-executes `build_*` functions for all invalidated downstream children against the updated `SceneContext`.
  - Re-triggers `/scene/finalize` (booleans + normalization).
  - Enforces a maximum revision loop budget of **3 cycles**.
- [ ] **Acceptance Criteria:** Adjusting the radius and height of `flask_body` causes downstream `neck` and `rim` to automatically re-snap flush to the new top boundary of `flask_body`.

---

#### Task 3.6: Orchestrator Test Suite (`tests/test_pipeline.py`)
- [ ] Implement mock integration and unit tests:
  - Test DAG topological ordering and cycle detection.
  - Test `SceneLedger` checkpoint addition, scoring rank, and anti-oscillation triggers.
  - Test fast-fail retry counter (verifying exactly 3 retries before failure on syntax error).
  - Test mock end-to-end loop: Plan $\rightarrow$ Code Chunks $\rightarrow$ Finalize $\rightarrow$ Revise $\rightarrow$ Re-anchor $\rightarrow$ Approve.
- [ ] **Acceptance Criteria:** `pytest tests/test_pipeline.py -v` passes with all tests green without requiring an active GPU.

---