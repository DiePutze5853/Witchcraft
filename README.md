1. System Architecture & High-Level Flow
This repository implements an autonomous, chunked 3D object generation pipeline. A fine-tuned multimodal model (Qwen/Qwen3.5-2B) translates natural language instructions into production-ready 3D meshes inside Blender using an end-to-end fluent Python Domain-Specific Language (DSL).
1.1 Core Principles
Pure 3D Volume Modeling: No 2D sketch arrays, raw vertex lists, or dynamic mesh face indices. Objects are built using 3D solid primitives, convex hulls, and Constructive Solid Geometry (CSG).
Canonical Working Units (CWU) & Final Normalization: The Coder authors geometry in local Canonical Working Units (
1
 CWU
≡
1
m
1 CWU≡1m
 pre-normalization, with root extents roughly 
0.5
−
1.0
 CWU
0.5−1.0 CWU
). During finalization, the entire composite assembly is scaled so its maximum axis extent equals exactly 
1.0
m
1.0m
.
Component-Wise Chunking & DAG Propagation: Multi-part objects are broken into a Directed Acyclic Graph (DAG) and coded one discrete component at a time. Patches to parent chunks automatically re-anchor downstream children.
Semantic Snapping: The Coder positions components flush against each other using .snap_to(), eliminating mental coordinate math.
Transactional Staging & Rollback: Blender executes code into an isolated staging buffer; any syntax, safety, or geometric defect triggers an immediate rollback.
Unified Fluent Syntax: Both the Coder and the Critic write fluent Python, maximizing token efficiency and model comprehension.
code
Code
┌────────────────────────┐
                              │ User Prompt / Intent   │
                              └───────────┬────────────┘
                                          │
                                          ▼
                         ┌─────────────────────────────────┐
                         │   1. SPATIAL PLANNER            │
                         │   [MODE: PLAN]                  │
                         │   Emits Component DAG & SSG     │
                         └────────────────┬────────────────┘
                                          │
                     ┌────────────────────┴────────────────────┐
                     │ Chunked Assembly Loop (Per Component)   │
                     ▼                                         ▼
         ┌───────────────────────┐                 ┌───────────────────────┐
         │ Component 1 (Root)    │                 │ Component N (Child)   │
         │ Coder -> Staging Exec │──(Commit)──────►│ Coder -> Staging Exec │
         └───────────────────────┘                 └───────────────────────┘
                     │                                         │
                     └────────────────────┬────────────────────┘
                                          │ All Chunks Staged
                                          ▼
                         ┌─────────────────────────────────┐
                         │   TRANSACTIONAL FINALIZATION    │
                         │   Sequential Booleans + Blame   │
                         │   Assembly Normalization (1.0m) │
                         └────────────────┬────────────────┘
                                          │
                                          ▼
                         ┌─────────────────────────────────┐
                         │   3. VISUAL & MESH CRITIC       │
                         │   [MODE: CRITIC]                │
                         │   Evaluates Strip + Geo Metrics │
                         └────────────────┬────────────────┘
                                          │
                     ┌────────────────────┴────────────────────┐
                     ▼                                         ▼
           [Patch: `revise()`]                        [Patch: `approve()`]
           (DAG Rebuild of Children)                           │
                     │                                         ▼
                     └──────► (Re-anchor Downstream)  [Export Verified .GLB]
                                                      [Roundtrip SSIM Render]
2. Agent Execution Contracts
The model is conditioned across three operational modes:
2.1 Mode 1: Spatial Planner ([MODE: PLAN])
Objective: Decompose the prompt into an ordered assembly DAG of named components.
Prohibitions: The Planner must not emit metric units or world coordinates.
Dimensionless Scale Convention: Component proportions are expressed using categorical classes and relative aspect ratios:
PRIMARY (Dominant structural mass)
SECONDARY (Supporting structures, attachments)
DETAIL (Fasteners, lips, trim, cuts)
Output Format: Strict Markdown layout tree.
code
Markdown
[MODE: PLAN]
TARGET: A fantasy potion flask

ASSEMBLY_ORDER:
1. flask_body (Root)
2. neck (Depends on: flask_body)
3. rim (Depends on: neck)
4. cavity (Depends on: neck, flask_body)

COMPONENTS:
- flask_body:
    primitive: Hull([Sphere, Sphere])
    scale_class: PRIMARY
    aspect_ratio: [1.0, 1.0, 1.4]
    geometry_notes: Wide lower sphere tapering smoothly into a smaller upper sphere.
    anchors: Root at origin

- neck:
    primitive: Cylinder
    scale_class: SECONDARY
    aspect_ratio: [0.3, 0.3, 0.8]
    geometry_notes: Slender vertical tube, ~1/3 the width of flask_body.
    anchors: RESTS_ON_CENTER -> flask_body.TOP

- rim:
    primitive: Torus
    scale_class: DETAIL
    aspect_ratio: [0.35, 0.35, 0.1]
    geometry_notes: Circular lip, slightly wider than neck outer radius.
    anchors: RESTS_ON_CENTER -> neck.TOP

- cavity:
    primitive: Cylinder
    operation: DIFFERENCE
    scale_class: SECONDARY
    aspect_ratio: [0.22, 0.22, 2.0]
    geometry_notes: Plunges down from neck top to hollow out flask interior.
    anchors: CARVED_INTO -> neck.TOP

MATERIAL:
- FlaskGlass (Color: (0.1, 0.7, 0.5), Roughness: 0.1, Alpha: 0.4)
2.2 Mode 2: DSL Chunk Coder ([MODE: CODE])
Objective: Translate a single component entry from the plan into an isolated, executable Python function: build_<chunk_id>(scene).
Rules:
Must strictly use exposed DSL classes and methods.
No external imports (import bpy, import os, etc. are blocked).
Use .snap_to() to attach components to existing scene parts.
Fast-Fail Gate: AST violations, syntax errors, or execution exceptions trigger up to 3 automated retries within [MODE: CODE] before failing the pipeline.
Example: Generating Chunk 1 (flask_body)
code
Python
# [MODE: CODE]
# Component: flask_body (Root)
# Bounds: z in [0.0, 0.80]
def build_flask_body(scene: SceneContext) -> Node3D:
    s_bottom = Sphere(r=0.35).at(z=0.35)
    s_top = Sphere(r=0.15).at(z=0.65)
    return Hull([s_bottom, s_top]).name("flask_body")
Example: Generating Chunk 2 (neck)
code
Python
# [MODE: CODE]
# Component: neck
# Context: "flask_body" is committed in scene (top at z=0.80).
# Bounds: z in [0.80, 1.05]
def build_neck(scene: SceneContext) -> Node3D:
    body = scene.get("flask_body")
    return Cylinder(r=0.08, h=0.25).snap_to(body, on="TOP").name("neck")
Example: Generating Chunk 4 (cavity)
code
Python
# [MODE: CODE]
# Component: cavity (Boolean Difference)
# Context: "flask_body" (z in [0.0, 0.80]) and "neck" (z in [0.80, 1.05]) exist.
# Cut height = 0.98. Snapping to neck TOP sets bottom at z=1.05.
# Moving dz = -0.98 aligns cutter top with neck rim (z=1.05) and bottom at z=0.07.
def build_cavity(scene: SceneContext) -> Node3D:
    neck = scene.get("neck")
    cutout = (
        Cylinder(r=0.06, h=0.98)
        .snap_to(neck, on="TOP")
        .move(dz=-0.98)
        .name("cavity")
    )
    return cutout.as_difference_for(["flask_body", "neck"])
2.3 Mode 3: Visual & Geometric Critic ([MODE: CRITIC])
Objective: Inspect composite multi-view diagnostic renders and geometric metadata, emitting fluent patch commands.
Input:
<image>: Combined horizontal strip (
1536
×
512
1536×512
 px) showing Front (ortho), Side (ortho), and Isometric (perspective) views.
Metadata string: Vertex/face counts, manifold status, bounding dimensions.
Output: A single-line statement using the Critic Patch DSL (Section 6).
Approval Case:
code
Python
approve(score=9.2, note="Proportions match reference, watertight manifold verified")
Revision Case:
code
Python
revise("neck").adjust(r=0.06, h=0.32).note("Neck is too short and thick for a delicate potion flask")
3. Domain-Specific Language (DSL) Specification
Implemented in core/dsl.py. Operates strictly on solid 3D volumes.
3.1 Transform Evaluation Order
Chained transformations on any Node3D primitive are evaluated deterministically:
Output Transform
=
Move
(
Δ
x
,
Δ
y
,
Δ
z
)
×
SnapTo
(
Target
,
Face
)
×
Rotate
(
θ
x
,
θ
y
,
θ
z
)
×
Scale
(
s
x
,
s
y
,
s
z
)
×
BaseMesh
Output Transform=Move(Δx,Δy,Δz)×SnapTo(Target,Face)×Rotate(θ 
x
​
 ,θ 
y
​
 ,θ 
z
​
 )×Scale(s 
x
​
 ,s 
y
​
 ,s 
z
​
 )×BaseMesh
.snap_to() establishes the base translation; .move() applies local delta offsets on top of the snapped position.
3.2 Complete API Reference Table
Class / Method	Signature	Return Type	Description
Box	Box(x: float, y: float, z: float)	Node3D	Centered box primitive. Dimensions 
>
0
>0
.
Cylinder	Cylinder(r: float, h: float, r_top: float = None)	Node3D	Cylinder. Optional r_top creates tapered frustum.
Cone	Cone(r_base: float, r_top: float = 0.0, h: float = 1.0)	Node3D	Circular cone or truncated cone.
Sphere	Sphere(r: float)	Node3D	UV sphere centered at origin.
Torus	Torus(major_r: float, minor_r: float)	Node3D	Torus lying in XY plane.
Hull	Hull(primitives: list[Node3D])	Node3D	Computes watertight convex hull wrapping inputs.
.at	.at(x: float = 0, y: float = 0, z: float = 0)	self	Sets absolute centroid translation.
.move	.move(dx: float = 0, dy: float = 0, dz: float = 0)	self	Applies relative delta offset.
.rotate	.rotate(x: float = 0, y: float = 0, z: float = 0)	self	Rotates by degrees around principal axes.
.scale	.scale(x: float = 1, y: float = 1, z: float = 1)	self	Non-uniform or uniform scaling. Factors 
>
0
>0
.
.snap_to	.snap_to(target: Node3D, on: SnapFace)	self	Snaps centroid flush to target face boundary.
.union	mesh_a.union(mesh_b: Node3D)	Node3D	CSG boolean union of two solid volumes.
.difference	mesh_a.difference(mesh_b: Node3D)	Node3D	Subtracts mesh_b volume from mesh_a.
.intersect	mesh_a.intersect(mesh_b: Node3D)	Node3D	Keeps only overlapping volume.
.as_difference_for	.as_difference_for(target_names: list[str])	Node3D	Tags node as subtraction mask for named chunks.
.displace	.displace(strength: float, roughness: float = 1.0)	self	Adds procedural surface noise for rocks/bark.
.bevel	.bevel(radius: float, segments: int = 2)	self	Rounds sharp edges. Clamped to prevent self-overlap.
.mirror	.mirror(axis: Literal['X', 'Y', 'Z'])	self	Mirrors geometry across the origin plane.
.material	.material(name: str, color: tuple, ...)	self	Assigns Principled BSDF material attributes.
.name	.name(chunk_id: str)	self	Registers node identifier in scene dictionary.
3.3 Semantic Snapping Rules (.snap_to)
Valid targets for on:
"TOP": Places bottom bounding plane of object flush against top bounding plane of target (
+
Z
+Z
). Centroid matches on 
X
,
Y
X,Y
.
"BOTTOM": Places top plane flush against bottom plane of target (
−
Z
−Z
).
"FRONT": Attaches flush to front bounding plane (
+
Y
+Y
).
"BACK": Attaches flush to back bounding plane (
−
Y
−Y
).
"RIGHT": Attaches flush to right bounding plane (
+
X
+X
).
"LEFT": Attaches flush to left bounding plane (
−
X
−X
).
4. Blender Execution & Safety Architecture
4.1 Dual AST Security Validators
To ensure robust parsing across Python versions (including Blender 4.2+ / Python 3.11+), static validation is split into two specialized whitelist visitors (security/validator.py):
code
Python
# security/validator.py
import ast

COMMON_ALLOWED_NODES = {
    ast.Module, ast.Constant, ast.Name, ast.Call, ast.keyword,
    ast.Attribute, ast.BinOp, ast.UnaryOp, ast.Add, ast.Sub,
    ast.Mult, ast.Div, ast.USub, ast.UAdd, ast.Load
}

class ChunkASTValidator(ast.NodeVisitor):
    """Validates component building functions: def build_<id>(scene): ..."""
    ALLOWED_NODES = COMMON_ALLOWED_NODES | {
        ast.FunctionDef, ast.arguments, ast.arg, ast.Return,
        ast.Assign, ast.List, ast.Tuple, ast.Store
    }
    ALLOWED_PRIMITIVES = {"Box", "Cylinder", "Cone", "Sphere", "Torus", "Hull"}

    def __init__(self, max_depth: int = 20):
        self.depth = 0
        self.max_depth = max_depth

    def generic_visit(self, node):
        self.depth += 1
        if self.depth > self.max_depth:
            raise SecurityError(f"AST recursion depth limit ({self.max_depth}) exceeded.")
        if type(node) not in self.ALLOWED_NODES:
            raise SecurityError(f"Disallowed syntax: {type(node).__name__}")
        super().generic_visit(node)
        self.depth -= 1

    def visit_Attribute(self, node):
        if node.attr.startswith("_"):
            raise SecurityError(f"Private/dunder access '{node.attr}' forbidden.")
        self.generic_visit(node)

    def visit_Call(self, node):
        # Allow Primitive instantiations: Box(...)
        if isinstance(node.func, ast.Name) and node.func.id in self.ALLOWED_PRIMITIVES:
            return self.generic_visit(node)
        # Allow scene.get("name")
        if (isinstance(node.func, ast.Attribute) and 
            isinstance(node.func.value, ast.Name) and 
            node.func.value.id == "scene" and node.func.attr == "get"):
            return self.generic_visit(node)
        # Allow method chaining on Node3D
        if isinstance(node.func, ast.Attribute) and hasattr(Node3D, node.func.attr):
            return self.generic_visit(node)
        raise SecurityError(f"Unauthorized call target: {ast.dump(node.func)}")

class PatchASTValidator(ast.NodeVisitor):
    """Validates top-level Critic expressions: approve(...) or revise(...)..."""
    ALLOWED_NODES = COMMON_ALLOWED_NODES | {ast.Expr}
    ALLOWED_ROOT_CALLS = {"approve", "revise"}
    ALLOWED_PATCH_METHODS = {"scale", "move", "adjust", "note"}

    def visit_Call(self, node):
        if isinstance(node.func, ast.Name) and node.func.id in self.ALLOWED_ROOT_CALLS:
            return self.generic_visit(node)
        if isinstance(node.func, ast.Attribute) and node.func.attr in self.ALLOWED_PATCH_METHODS:
            return self.generic_visit(node)
        raise SecurityError(f"Unauthorized Critic call: {ast.dump(node.func)}")
Chunk execution runs inside an isolated subprocess monitored by a 5-second hard execution watchdog timer.
4.2 Transactional Staging & Rollback
Chunks are isolated into an internal Blender staging collection during execution:
Staging Execution: The chunk's function executes within a temporary collection (__staging__).
Intermediate Audit: The chunk geometry is verified against the geometric acceptance suite (Section 5).
Commit: If valid, the object is moved from __staging__ to __active_scene__, registering its bounding box and metadata in scene.
Rollback: If validation fails, __staging__ is purged. The existing scene state remains clean.
4.3 Transactional Boolean Finalization & Blame Attribution
During /scene/finalize, boolean operations are executed from a dependency queue with isolated rollback checkpoints:
Per-Operation Snapshots: Before each boolean cut, the target mesh datablock is cloned as a backup snapshot.
Sequential Execution: Operations are executed one at a time.
Failure Attribution: If a boolean operation produces a non-manifold mesh, self-intersections, or an engine crash:
The target geometry is restored immediately from its snapshot.
The failure is mapped directly to the offending cutter chunk.
An automated recovery pass applies the adaptive epsilon offset (
±
ϵ
±ϵ
). If that fails, the failure is returned to [MODE: CODE] with targeted diagnostic logs.
Assembly Normalization: Once all booleans pass, the entire composite assembly is centered and uniformly scaled to fit exactly within a 1.0-meter bounding cube:
Scale Factor
=
1.0
max
⁡
(
Δ
X
,
Δ
Y
,
Δ
Z
)
Scale Factor= 
max(ΔX,ΔY,ΔZ)
1.0
​
 
5. Geometry & Export Acceptance Suite
Every chunk and finalized assembly must pass this validation suite:
Watertight 2-Manifold Check: Every edge must connect to exactly two faces (edge.is_manifold == True). Open boundaries or non-manifold edges are rejected.
Degenerate Faces: Zero tolerance for zero-area faces (
Area
<
10
−
6
 CWU
2
Area<10 
−6
  CWU 
2
 
).
Normal Consistency: Face normals must point consistently outward from the solid material volume (exterior faces point outward into world space; interior cavity faces point inward toward the empty void).
Self-Intersection Check: Meshes must be free of self-intersecting coplanar faces (verified via trimesh.intersections).
Adaptive Boolean Tolerance Policy: Computed using the minimum bounding dimension of the two participating boolean operands (
A
A
 and 
B
B
):
ϵ
=
clamp
(
min
⁡
(
dim
min
⁡
(
A
)
,
dim
min
⁡
(
B
)
)
×
0.005
,
0.0005
,
0.005
)
ϵ=clamp(min(dim 
min
​
 (A),dim 
min
​
 (B))×0.005,0.0005,0.005)

Cutter volumes are expanded outward along the cut normal by 
ϵ
ϵ
 before re-executing.
glTF Material & Schema Audit: The exported .glb is parsed to confirm proper serialization of PBR metallic-roughness factors, alpha modes (OPAQUE vs BLEND), and node hierarchies.
Roundtrip Render Verification: The exported .glb is re-imported into an isolated verification scene and rendered headlessly. Structural Similarity Index (SSIM) against the pre-export working render must meet:
SSIM
(
Render
Blender
,
Render
GLB
)
≥
0.92
SSIM(Render 
Blender
​
 ,Render 
GLB
​
 )≥0.92
6. Critic Patch DSL & State Ledger
6.1 Patch Grammar
The Critic emits concise fluent Python patch expressions:
code
Python
# Pass
approve(score: float, note: str)

# Revision
revise(target_chunk: str) \
    .scale(x: float = 1.0, y: float = 1.0, z: float = 1.0) \
    .move(dx: float = 0.0, dy: float = 0.0, dz: float = 0.0) \
    .adjust(**kwargs) \
    .note(reason: str)
Semantics:
.scale(...) operations are multiplicative (
x
new
=
x
old
×
s
x
x 
new
​
 =x 
old
​
 ×s 
x
​
 
).
.move(...) operations are additive (
z
new
=
z
old
+
d
z
z 
new
​
 =z 
old
​
 +dz
).
.adjust(...) updates named primitive parameters directly (e.g., r=0.06, h=0.32).
6.2 DAG-Aware Patch Rebuild Propagation
When a revision targets a component 
C
C
:
Target Update: Apply patch transformations to component 
C
C
.
DAG Invalidation: Identify all topological descendants of 
C
C
 in the Planner assembly graph.
Automatic Re-Anchoring: Re-execute the deterministic build_* functions for all downstream descendants. Because child chunks use .snap_to(), they automatically snap flush to 
C
C
's updated boundary positions.
Finalization Re-run: Re-execute boolean cuts and assembly normalization.
6.3 State Ledger & Best-Valid-State Recovery
Ledger Checkpointing: The orchestrator maintains a versioned ledger of every verified scene state, pairing geometry snapshots with Critic aesthetic scores.
Anti-Oscillation Detection: If opposite adjustments on the same chunk parameter occur across consecutive turns (e.g., 
s
z
=
1.25
s 
z
​
 =1.25
 followed by 
s
z
=
0.8
s 
z
​
 =0.8
), oscillation is flagged.
Fail-Closed Resolution: When oscillation is detected or the 3-loop critique budget expires:
The pipeline rolls back to the highest-scoring verified valid checkpoint in the ledger.
If no checkpoint has passed all mandatory geometric tests (Section 5), the pipeline triggers an explicit failure. It never forces the export of an unverified or broken mesh.
7. Unsloth Fine-Tuning Pipeline (Qwen/Qwen3.5-2B)
7.1 Synthetic Data Flywheel
code
Code
┌─────────────────────────────────────────┐
│ Frontier Model (Claude 3.5 Sonnet)      │
└───────────────────┬─────────────────────┘
                    │ Generates 3,000+ DAG Plans & DSL Chunk Scripts
                    ▼
┌─────────────────────────────────────────┐
│ Blender Daemon Automated Test Bench     │
└───────────────────┬─────────────────────┘
                    │ Filters syntax failures; renders 3-view diagnostic strips
                    ▼
┌─────────────────────────────────────────┐
│ Synthetic Visual Critic Generation      │
└───────────────────┬─────────────────────┘
                    │ Triplet Mining: (Prompt, Stitched Render, Flawed Chunk)
                    │ ──► Emits Critic Patch DSL ──► Emits Fixed Chunk DSL
                    ▼
┌─────────────────────────────────────────┐
│ Unsloth Multi-Task Training Set         │
└─────────────────────────────────────────┘
7.2 Training Configuration (training/train.py)
Following Unsloth fine-tuning guidelines for Qwen3.5-2B:
code
Python
# Unsloth Training Hyperparameters
BASE_MODEL = "unsloth/Qwen3.5-2B"      # or "Qwen/Qwen3.5-2B"
LOAD_IN_4BIT = False                  # bf16 LoRA avoids hybrid quantization degradation
BFLOAT16 = True                       # ~5GB VRAM footprint during training

LORA_RANK = 16
LORA_ALPHA = 32
LORA_DROPOUT = 0.05
TARGET_MODULES = [
    "q_proj", "k_proj", "v_proj", "o_proj",
    "gate_proj", "up_proj", "down_proj"
]

MAX_SEQ_LENGTH = 4096
LEARNING_RATE = 2e-4
LR_SCHEDULER_TYPE = "cosine"
WARMUP_RATIO = 0.10
OPTIMIZER = "adamw_8bit"
BATCH_SIZE = 2
GRADIENT_ACCUMULATION_STEPS = 8       # Effective batch size = 16
WEIGHT_DECAY = 0.01
RANDOM_SEED = 3407
7.3 Loss Masking Protocol
Cross-entropy loss is computed strictly on assistant tokens (<|im_start|>assistant to <|im_end|>). User prompts, system instructions, and <image> visual embedding tokens are masked out with label = -100.
8. Evaluation Benchmarks & Quantitative Targets
Checkpoints are evaluated against a held-out benchmark of 200 prompts divided evenly across four asset families: Mechanical Props, Stylized Containers, Architectural Furniture, and Fantasy Weapons. Metrics report 95% Wilson score confidence intervals.
Metric Category	Metric	Baseline (Zero-Shot)	Fine-Tuned Target	Verification Method
Security	System Security Enforcement	
100.0
%
100.0%
100.0
%
100.0%
Dual AST whitelist gateway. Zero escapes.
Model Quality	Model AST Compliance Rate	
82.0
%
±
5.3
%
82.0%±5.3%
≥
97.0
%
±
2.4
%
≥97.0%±2.4%
First-pass compliance with AST whitelist.
Compilation	Chunk Syntax Pass@1	
64.5
%
±
6.6
%
64.5%±6.6%
≥
95.0
%
±
3.0
%
≥95.0%±3.0%
Clean execution on first attempt.
Geometry	2-Manifold Watertight Rate	
51.0
%
±
6.9
%
51.0%±6.9%
≥
92.0
%
±
3.8
%
≥92.0%±3.8%
trimesh.is_watertight == True on export.
Geometry	Degenerate Face Rate	
14.2
%
±
4.8
%
14.2%±4.8%
≤
1.0
%
±
1.4
%
≤1.0%±1.4%
Faces with surface area 
<
10
−
6
 CWU
2
<10 
−6
  CWU 
2
 
.
Convergence	Mean Iterations to Approval	
3.4
±
0.4
3.4±0.4
≤
1.8
±
0.2
≤1.8±0.2
Revision cycles before Critic approval.
Convergence	Oscillation Rate	
22.0
%
±
5.7
%
22.0%±5.7%
≤
4.0
%
±
2.7
%
≤4.0%±2.7%
Frequency of contradictory patch updates.
Rendering	GLB Roundtrip SSIM 
≥
0.92
≥0.92
78.0
%
±
5.7
%
78.0%±5.7%
≥
98.0
%
±
1.9
%
≥98.0%±1.9%
Re-imported render similarity audit.
