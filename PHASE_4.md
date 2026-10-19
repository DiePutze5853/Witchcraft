# Phase 4 — Sprint 4: Geometric Acceptance Suite & Export Verification

**Objective:** Implement the strict geometric audit and export verification suite (`validation/`). This system verifies that meshes staged or finalized by the runtime meet production standards (watertight 2-manifold, zero degenerate faces, correct outward normal orientation, absence of self-intersections), audits exported `.glb` files against glTF PBR standards, and executes a headless render roundtrip audit enforcing $\text{SSIM} \ge 0.92$.

---

### Sprint 4 Task Breakdown

#### Task 4.1: Watertight 2-Manifold & Boundary Audit (`validation/geometry.py`)
- [ ] Implement `check_watertight_manifold(mesh: trimesh.Trimesh) -> ValidationResult`:
  - Enforce `mesh.is_watertight == True`.
  - Validate that every edge connects to exactly two faces (`edge.is_manifold == True`).
  - Detect and reject boundary edges (open holes, unclosed meshes) and non-manifold edges/vertices (edges shared by $\ge 3$ faces, bow-tie vertices).
  - Compute Euler characteristic $\chi = V - E + F$ and verify consistency with expected topological genus.
- [ ] Return structured diagnostic reports containing failing edge/vertex indices to feed back into code repair loops.
- [ ] **Acceptance Criteria:** Reject any mesh with open boundaries or non-manifold topology; pass watertight solid shapes.

---

#### Task 4.2: Normal Consistency & Cavity Orientation Audit (`validation/geometry.py`)
- [ ] Implement `check_face_normals(mesh: trimesh.Trimesh) -> ValidationResult`:
  - Verify global winding order consistency across all connected face components.
  - Calculate signed volume ($\text{volume} > 0$ indicates valid exterior winding).
  - Implement Solid-Relative Outward Normals verification:
    - Exterior hull faces must point outward into world space.
    - Interior hollowed-out cavity faces (generated via `DIFFERENCE` booleans) must point inward toward the empty internal void (away from the solid material wall).
  - Detect flipped, zero-length, or divergent normal vectors.
- [ ] **Acceptance Criteria:** Inverted winding orders and incorrectly flipped cavity normals trigger an immediate validation failure.

---

#### Task 4.3: Degenerate Geometry & Self-Intersection Audit (`validation/geometry.py`)
- [ ] Implement `check_degenerate_faces(mesh: trimesh.Trimesh, area_threshold: float = 1e-6) -> ValidationResult`:
  - Inspect all triangular face areas.
  - Flag any face where:
    $$\text{Area} < 10^{-6}\text{ CWU}^2$$
  - Flag collinear edges, duplicate vertices, and unreferenced/floating vertices.
- [ ] Implement `check_self_intersections(mesh: trimesh.Trimesh) -> ValidationResult`:
  - Run `trimesh.intersections.mesh_self_intersection(mesh)` to detect overlapping coplanar or interpenetrating faces within the single consolidated body.
  - Reject models with self-intersecting hulls or internal orphaned faces resulting from unmerged boolean operations.
- [ ] **Acceptance Criteria:** Meshes containing sliver faces ($< 10^{-6}$) or self-intersecting surfaces fail the audit with exact triangle indices logged.

---

#### Task 4.4: glTF/GLB Schema & Material Audit (`validation/export_audit.py`)
- [ ] Implement `audit_glb_schema(glb_path: Path) -> ValidationResult`:
  - Parse binary `.glb` header, chunks, and JSON manifest.
  - Verify adherence to the glTF 2.0 specification.
  - Confirm node hierarchy consistency and correct transform matrices.
- [ ] Implement PBR Material Attribute Inspection:
  - Check that materials use standard PBR metallic-roughness factor channels.
  - Verify alpha mode serialization: confirm translucent materials (e.g. glass with $\alpha < 1.0$) serialize with `alphaMode: "BLEND"` or `"MASK"`, and opaque materials serialize with `alphaMode: "OPAQUE"`.
  - Ensure texture coordinates (`TEXCOORD_0`) are valid if UV channels are present.
- [ ] **Acceptance Criteria:** Exported `.glb` is parsed and confirmed compliant with the glTF 2.0 schema and specified PBR material parameters.

---

#### Task 4.5: Headless Multi-View & Roundtrip SSIM Render Verification (`validation/export_audit.py`)
- [ ] Implement `render_pre_export_strip(blender_client, out_path: Path) -> Path`:
  - Render pre-export working geometry in Blender headlessly using a calibrated 3-camera horizontal strip rig ($1536 \times 512$ px: Front ortho, Side ortho, Isometric perspective).
- [ ] Implement `render_glb_roundtrip_strip(glb_path: Path, out_path: Path) -> Path`:
  - Spin up an isolated, clean Blender verification scene.
  - Import the exported `.glb` binary.
  - Render using the identical camera rig, lighting setup, and resolution ($1536 \times 512$ px).
- [ ] Implement Structural Similarity Index (SSIM) Audit:
  - Load pre-export render and imported `.glb` render into memory via `scikit-image` (`skimage.metrics.structural_similarity`).
  - Compute multi-channel SSIM across the image strip:
    $$\text{SSIM}(\text{Render}_{\text{Blender}}, \text{Render}_{\text{GLB}}) \ge 0.92$$
  - If $\text{SSIM} < 0.92$: Reject export, flag discrepancy (missing geometry, material conversion failure, or normal inversion), and fail the pipeline run.
- [ ] **Acceptance Criteria:** Discrepancies between internal Blender rendering and exported `.glb` rendering that score below $0.92$ SSIM trigger an automated failure.

---

#### Task 4.6: End-to-End Integration Test Suite (`tests/test_integration.py`)
- [ ] Implement full end-to-end integration test (`tests/test_integration.py`):
  1. Define a sample multi-part prompt (e.g., the fantasy potion flask from Section 2.1).
  2. Parse the plan through `AssemblyDAG`.
  3. Execute chunk builders into `runtime/staging.py`.
  4. Trigger transactional boolean finalization and $1.0\text{m}$ normalization.
  5. Run all geometric audits in `validation/geometry.py`.
  6. Export verified `.glb`.
  7. Run `audit_glb_schema` and verify $\text{SSIM} \ge 0.92$.
- [ ] Verify fail-closed behavior:
  - Inject an intentional open boundary or degenerate sliver; verify export is aborted and `PipelineFailureError` is raised.
- [ ] **Acceptance Criteria:** `pytest tests/test_integration.py -v` executes the entire pipeline from plan ingestion to verified `.glb` export and passes with 100% green status.

---