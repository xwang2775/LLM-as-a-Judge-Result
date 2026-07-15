# Exp2 — Sub-Category (11-way) Agreement, and Explicit Granularity Statement

> **给执行 agent 的说明（中文）**：这是一个**小实验**，用来回应审稿人 Comment 4（"请确认 κ 是在 7 个 top-level category 上算的，并说明是否也在 11 个 subcategory 上算过"）。**目前作者只在 7 类上算了 κ，没有在 11 个子类上算过。** 这个实验的目标是：若已有标注里能拿到子类标签，就补算 11 类的 Cohen's κ（H–H 和 H–J 各一个）；若拿不到子类标签，就明确记录"无法回溯计算"，并给出一个**低成本的补标方案**（同一批 40 条，重复用同一 rubric 打子类标签）。不需要跑 agent，不需要新采样。

---

## 0. Background
The paper reports inter-annotator $\kappa=0.61$ (H–H) and human–judge $\kappa=0.84$ (H–J) at the **7 top-level category** granularity (main text §3; Appendix `app:4` explicitly says "Both are reported at the same 7-category granularity"). The taxonomy also has **11 subcategories** (Appendix `app:4`, "Taxonomy complexity": "7 top-level categories with 11 subcategories"). The reviewer asks whether κ was **also** computed at the 11-subcategory level.

**Ground truth to state in the rebuttal (already confirmed by the authors):** κ was computed **only** on the 7 top-level categories; it was **not** computed on the 11 subcategories.

## 1. Inputs (locate first)
- The **two 40-trajectory validation sets** (the H–H set with two annotators; the H–J set with a third annotator + judge). Fields: `trajectory_id`, `domain`, `annotator_label`, `judge_label`, and — critically — **whether subcategory labels were recorded**.
- The mapping from **11 subcategories → 7 top-level categories** (from Appendix `app:1` per-domain definitions). Print it and confirm it is exactly 11 → 7.

## 2. Task
**Case A — subcategory labels already exist in the stored annotations:**
1. Compute **Cohen's $\kappa$ on the 11 subcategories** for H–H and for H–J, at the **same unit of analysis** as the reported 7-category κ (per-step vs. per-trajectory-primary-label — match exactly; record it).
2. Report a **confusion matrix** over the 11 subcategories (where disagreements concentrate — expect within-parent confusions, e.g. sub-plan vs. action inside Planning Error).
3. Recompute the **7-category κ from the same data** as a consistency check (must match the reported 0.61 / 0.84 within ±0.02).

**Case B — subcategory labels were NOT recorded (most likely, per the authors):**
1. Do **not** fabricate. Report that subcategory κ cannot be computed retroactively.
2. Provide a **lightweight re-labeling protocol** the authors can run on the *same* 40-trajectory sets: one rubric-trained annotator + the judge re-emit the 11-subcategory label for each already-localized failure step (no new sampling, no new trajectories, reuse the recorded failure step). Estimate effort (≈ a few hours per set, since trajectories are already read).
3. Produce the empty CSV/`.tex` templates so results can be dropped in if the authors choose to run the re-labeling.

## 3. Deliverables
- `subcategory_kappa.csv`: `study(H-H|H-J), granularity(7|11), unit, n, cohen_kappa, kappa_ci_low, kappa_ci_high` (+ bootstrap 95% CI, B=10000, seed recorded).
- `subcategory_confusion.pdf` (11×11) if Case A.
- `subcategory_kappa_table.tex` for `app:4`.
- 2–3 sentences for the rebuttal stating: 7-category κ = 0.61/0.84 (confirmed granularity); subcategory κ = [value] (Case A) **or** "not computed; would require re-labeling the same 40 traces, which we can add" (Case B).

## 4. Expected result / framing
- If Case A: subcategory κ will be **somewhat lower** than 7-category κ (finer classes → more boundary confusion). This is **expected and normal**; frame it as "agreement remains substantial at top level; residual disagreement is concentrated in within-parent subcategory boundaries, which is why we report and analyze at the 7-category level."
- If Case B: the honest statement ("computed on 7, not 11") **fully answers** Comment 4 as asked; the re-labeling is optional value-add.

## 5. Acceptance Criteria
- [ ] Unit of analysis identical to the reported 0.61/0.84; documented.
- [ ] 7-category κ recomputed from the same data matches the paper (±0.02) before reporting any 11-way number.
- [ ] No fabricated subcategory labels; Case B handled honestly.
- [ ] Every κ has `n` and a bootstrap 95% CI.
