# Exp3 — Judge Validation Table: Per-Category Precision / Recall / F1 + 7×7 Confusion Matrix

> **给执行 agent 的说明（中文）**：这是一个**纯分析实验，不需要重跑任何 agent，也不需要重新调用 LLM judge**。全部基于**已有的 40 条 human–judge (H–J) 校准集**（即产生 human–judge $\kappa=0.84$ 的那批标注）。目标是把审稿人 Comment "Add a judge validation table with per-category precision/recall/F1 and a confusion matrix" 从"未回应"变成"已量化"。把**人类标签当作 reference（gold）**，**judge 标签当作 prediction**，算出每个 category 的 precision/recall/F1，以及一个 7×7（或含 Others 的 8×8）confusion matrix。请严格按 §6 Acceptance Criteria 自查后再交付。若数据 schema 与本文假设不同，**先读真实数据、打印字段，再对齐**，并在报告里记录实际用的字段与单位。

---

## 0. Background & Motivation

**Paper.** *Why Do Long-Horizon Agents Fail?* validates its trajectory-grounded LLM-as-a-Judge against expert humans on a **40-trajectory H–J calibration set**, reporting only a single aggregate agreement number (human–judge Cohen's $\kappa=0.84$; Appendix `app:4`). The judge assigns **exactly one** of 7 categories (+ `Others`) per trajectory — the root cause at the earliest critical failure step (judge prompt, Appendix `app:4`, lines ~3026–3115).

**Reviewer concern (verbatim intent).** "Add a judge validation table with per-category precision/recall/F1 and a confusion matrix." A single aggregate $\kappa$ hides *which* categories the judge is reliable on and *where* it systematically confuses categories. The main conclusions ("three categories dominate": Planning Error, Memory Limitation, Catastrophic Forgetting) depend on the judge, so per-category reliability must be shown for exactly those categories.

**Goal.** Produce a per-category reliability table + confusion matrix from **existing** H–J labels, so the judge's reliability can be read category-by-category (especially for the three dominant categories).

---

## 1. Inputs (locate these first)

The LaTeX-only folder here does **not** contain the data. Locate the real artifacts (sibling results/benchmark repo). You need:

### 1.1 The 40-trajectory H–J validation set
The **same** set that produced $\kappa=0.84$. Expected fields (names may differ — map and record the mapping):
- `trajectory_id`, `domain` ∈ {Web, OS, Embodied, Database}
- `human_label` — one of the 7 categories (+ `Others`), at the **same unit** used to compute 0.84 (per-trajectory primary label is expected, since the judge emits one label per trajectory).
- `judge_label` — the judge's `failure_type` for the same trajectory/unit.

> **Critical:** locate the existing κ-computation script and **match its unit of analysis exactly** (per-trajectory primary label; label set = 7 vs. 8 with `Others`). Record the unit. Do **not** invent a per-step alignment if the reported 0.84 was per-trajectory.

---

## 2. Task

Treat **human = reference (gold)**, **judge = prediction** (state this convention explicitly; it is the standard framing for "is the judge faithful to the human/rubric").

1. **Recompute the aggregate $\kappa$ from this same data first** as a wiring sanity check — must match the reported 0.84 within ±0.02 before reporting anything else. If it does not match, **stop and report the discrepancy** (do not proceed on mismatched data).
2. **Confusion matrix** (rows = human/gold category, cols = judge/predicted category), 7×7 (and an 8×8 variant if `Others` is used). Include raw counts; also a row-normalized version (recall view).
3. **Per-category metrics**, for each of the 7 (+ `Others`) categories:
   - `precision` = judge-correct / all-judge-predicted-this-category
   - `recall` = judge-correct / all-human-labeled-this-category
   - `f1` = harmonic mean
   - `support` = # human-labeled trajectories in that category
4. **Aggregate summaries:** macro-F1, micro-F1 (== accuracy for single-label), weighted-F1, and overall accuracy.
5. **Uncertainty:** bootstrap 95% CI (resample trajectories with replacement, B=10000, fixed seed recorded) on macro-F1 and on the F1 of each of the three dominant categories. Per-category `support` on 40 traces is small (often <10), so CIs will be wide — **report them honestly** rather than presenting per-category F1 as precise.

---

## 3. Deliverables

- `Exp3_judge_confusion.csv` — raw 8×8 counts (human rows × judge cols).
- `Exp3_per_category_prf1.csv`: `category, support, precision, recall, f1, f1_ci_low, f1_ci_high`.
- `Exp3_confusion_matrix.pdf` — heatmap (row-normalized), for `app:4`.
- `Exp3_prf1_table.tex` — LaTeX table (per-category P/R/F1 + macro/micro/weighted F1 + accuracy) for `app:4`.
- 3–4 sentences for the rebuttal reporting: aggregate $\kappa$ reproduced (= 0.84 ±0.02), macro-F1 [CI], and the F1 of the three dominant categories, plus the main off-diagonal confusion (expected: Q5 Catastrophic Forgetting ↔ Q6 Memory Limitation, and Planning Error ↔ False Assumption, since these are the rubric's closest neighbors).

---

## 4. Expected result / framing

- The **three dominant categories (Planning Error, Memory Limitation, Catastrophic Forgetting)** should show the highest support and solid F1 — this is the key point: the categories the paper's conclusions rest on are exactly the ones the judge labels most reliably.
- **Expected confusion:** the largest off-diagonal mass should be **within-rubric-neighbor** confusions — Memory Limitation ↔ Catastrophic Forgetting (Q5/Q6 differ only by whether info is still in-context; the prompt itself flags this as the hard boundary, lines ~3033–3037), and Planning Error ↔ False Assumption (Q3/Q4). Frame this as *interpretable, boundary-consistent* disagreement, not random error.
- If a **minor** category has near-zero support/F1 on 40 traces, say so plainly and note it does not affect the dominant-category conclusion; it is a target for the expanded human batch (cross-reference W5 §5).

---

## 5. Contingency

- If `human_label`/`judge_label` are stored per-step rather than per-trajectory, compute the table at the **per-step** unit but **clearly label it**, and additionally collapse to per-trajectory primary label to match the reported 0.84. Report which unit the 0.84 used.
- If `Others` is degenerate (0 support), report the 7×7 and note `Others` was empty in this sample.

---

## 6. Acceptance Criteria (self-check before delivering)

- [ ] Aggregate $\kappa$ recomputed from this data matches the reported 0.84 (±0.02) **before** any P/R/F1 is reported.
- [ ] Human = gold, judge = prediction convention stated explicitly.
- [ ] Unit of analysis (per-trajectory primary vs. per-step) documented and matched to the reported κ.
- [ ] Every per-category F1 has a `support` and a bootstrap 95% CI (B≥10000, seed recorded).
- [ ] Confusion matrix sums to n (=40) and is provided as raw counts + row-normalized.
- [ ] No re-running of agents or the LLM judge; this is analysis of existing labels only.
- [ ] Report flags any category whose small support makes its F1 uninformative (honesty over optimism).

---

## 7. Mapping to the rebuttal

Feeds: "Per-reviewer request, we add a judge-validation table (Table X, `app:4`) reporting per-category precision/recall/F1 with bootstrap CIs and a 7×7 confusion matrix on the 40-trajectory H–J set. The three categories our conclusions rely on (Planning Error, Memory Limitation, Catastrophic Forgetting) are the highest-support, highest-F1 categories; residual confusion concentrates on the rubric-adjacent Q5/Q6 (Catastrophic Forgetting vs. Memory Limitation) boundary, consistent with our decision-tree design (Fig. `fig:decision_tree`). No agents or judge re-runs were required."
