# Phase 5 — Sprint 5: Synthetic Data Flywheel, Unsloth Fine-Tuning & Benchmark Harness

**Objective:** Implement the dataset synthesis flywheel, prompt templates, Unsloth multimodal fine-tuning pipeline for `Qwen/Qwen3.5-2B` (bf16 LoRA with strict assistant token loss masking), and the quantitative 200-prompt evaluation benchmark harness reporting 95% Wilson score confidence intervals.

---

### Sprint 5 Task Breakdown

#### Task 5.1: Multi-Mode Prompt System Contracts (`training/prompts.py`)
- [ ] Implement system prompt definitions enforcing exact output templates:
  - `MODE_PLAN_SYSTEM_PROMPT`: Directs the model to output the strict Markdown layout tree (`ASSEMBLY_ORDER`, `COMPONENTS` with `primitive`, `scale_class`, `aspect_ratio`, `geometry_notes`, `anchors`, and `MATERIAL`). Strict prohibition against hardcoded metric dimensions.
  - `MODE_CODE_SYSTEM_PROMPT`: Restricts output to `def build_<chunk_id>(scene: SceneContext) -> Node3D:`. Strictly enforces fluent DSL primitives, `.snap_to()`, and forbids `bpy` or external imports.
  - `MODE_CRITIC_SYSTEM_PROMPT`: Instructs the vision-language model to ingest the $1536 \times 512$ 3-view strip and output a single-line patch command: either `approve(...)` or `revise("<chunk>")...`.
- [ ] **Acceptance Criteria:** Formatting helper functions produce correctly formatted ChatML sequences (`<|im_start|>system...`, `<|im_start|>user...`, `<|im_start|>assistant...`).

---

#### Task 5.2: Synthetic Data Flywheel & Triplet Mining (`training/flywheel.py`)
- [ ] Implement Teacher LLM Orchestrator:
  - Generate 3,000+ candidate multi-part assembly plans and corresponding chunk scripts across 4 asset families: Mechanical Props, Stylized Containers, Architectural Furniture, and Fantasy Weapons.
- [ ] Implement Automated Blender Testbench Filter:
  - Feed synthetic scripts into `runtime/blender_server.py`.
  - Discard any candidates that fail `ChunkASTValidator` or throw runtime exceptions.
  - Render calibrated $1536 \times 512$ diagnostic multi-view strips (Front ortho, Side ortho, Iso perspective) for all structurally valid models.
- [ ] Implement Triplet Mining for Critic Training:
  - Systematically inject controlled perturbations into valid chunks (e.g., scale deviations, incorrect snap faces, oversized/undersized radii).
  - Render flawed state strip.
  - Form training triplets: `(Prompt + Flawed Strip, Corrective Critic Patch DSL, Fixed Chunk DSL)`.
  - Package final multi-task dataset into Unsloth-compatible JSONL format.
- [ ] **Acceptance Criteria:** Flywheel executes unattended, filters out broken syntax, and emits verified (image, plan, chunk, patch) training samples.

---

#### Task 5.3: Training Configuration & Hyperparameters (`training/configs/qwen35_bf16.yaml`)
- [ ] Create YAML configuration specifying the exact hyperparameter contract:
  ```yaml
  base_model: "unsloth/Qwen3.5-2B"
  load_in_4bit: false # bf16 LoRA avoids hybrid quantization degradation
  bfloat16: true # ~5GB VRAM footprint
  lora:
    rank: 16
    alpha: 32
    dropout: 0.05
    target_modules:
      - "q_proj"
      - "k_proj"
      - "v_proj"
      - "o_proj"
      - "gate_proj"
      - "up_proj"
      - "down_proj"
  training:
    max_seq_length: 4096
    learning_rate: 0.0002
    lr_scheduler_type: "cosine"
    warmup_ratio: 0.10
    optimizer: "adamw_8bit"
    batch_size: 2
    gradient_accumulation_steps: 8 # Effective batch size = 16
    weight_decay: 0.01
    seed: 3407
  ```
- [ ] **Acceptance Criteria:** Configuration loader validates all fields against strict Pydantic schemas.

---

#### Task 5.4: Unsloth Fine-Tuning Engine & Loss Masking (`training/train.py`)
- [ ] Implement fine-tuning runner using `unsloth.FastVisionModel` / `FastLanguageModel`:
  - Load `unsloth/Qwen3.5-2B` with `load_in_4bit = False` and `bfloat16 = True`.
  - Attach PEFT LoRA adapters targeting all attention and MLP projection layers.
- [ ] Implement Strict Assistant Loss Masking:
  - Tokenize conversations ensuring `<image>` visual embeddings, system instructions, and user prompt tokens are set to `label = -100`.
  - Calculate cross-entropy loss exclusively on completion tokens generated between `<|im_start|>assistant` and `<|im_end|>`.
- [ ] Add checkpoint export utilities saving merge-ready LoRA adapter weights and HuggingFace-compatible configs.
- [ ] **Acceptance Criteria:** Training script trains within the ~5GB VRAM ceiling, verifies that prompt and vision tokens have 0 gradient contribution (`label = -100`), and converges with decreasing validation loss.

---

#### Task 5.5: Quantitative Benchmark & Evaluation Harness (`benchmarks/evaluate.py`)
- [ ] Curate held-out test suite of 200 prompts:
  - 50 Mechanical Props, 50 Stylized Containers, 50 Architectural Furniture, 50 Fantasy Weapons.
- [ ] Implement automated benchmark runner measuring performance against targets with 95% Wilson score confidence intervals:
  - **Security Enforcement:** Target $100.0\%$ (zero AST escapes).
  - **Model AST Compliance Rate:** Target $\ge 97.0\% \pm 2.4\%$.
  - **Chunk Syntax Pass@1:** Target $\ge 95.0\% \pm 3.0\%$.
  - **2-Manifold Watertight Rate:** Target $\ge 92.0\% \pm 3.8\%$ (`trimesh.is_watertight`).
  - **Degenerate Face Rate:** Target $\le 1.0\% \pm 1.4\%$ ($\text{Area} < 10^{-6}\text{ CWU}^2$).
  - **Mean Iterations to Approval:** Target $\le 1.8 \pm 0.2$.
  - **Oscillation Rate:** Target $\le 4.0\% \pm 2.7\%$.
  - **GLB Roundtrip SSIM $\ge 0.92$:** Target $\ge 98.0\% \pm 1.9\%$.
- [ ] Export summary tables in Markdown and JSON formats.
- [ ] **Acceptance Criteria:** `python -m benchmarks.evaluate --checkpoint <path>` runs the full 200-prompt evaluation headless pipeline and outputs the Wilson confidence interval scorecard.

---

#### Task 5.6: Final System Hand-off & End-to-End Verification
- [ ] Perform complete repo audit:
  - Run `ruff check .` and `mypy core security pipeline training validation`.
  - Run full test suite: `pytest -v`.
  - Verify strict `bpy` boundary enforcement across non-runtime packages.
  - Verify that execution in `runtime/` respects the 5-second watchdog timer, transactional staging buffer, and blame-attributed sequential booleans.
- [ ] **Acceptance Criteria:** All unit, security, runtime, and integration tests pass cleanly.

---