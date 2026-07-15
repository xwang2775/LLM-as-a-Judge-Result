# Exp4 — Single-Label Attribution (Clarification) + Optional Multi-Label Co-occurrence

> **给执行 agent 的说明（中文）**：这个文档回应两条审稿意见：(a) "Clarify whether failure attribution is single-label, multi-label"，(b) "the main failure distribution may not fully reflect multi-label failure composition"。**先讲清楚一个关键事实（已从论文确认）**：本文的 judge pipeline **是 single-label 的**——每条轨迹只输出一个 `failure_type`，即"最早关键失败步的 root cause"（judge prompt 在 `app:4`，约 3026–3115 行；输出 JSON 只有一个 `failure_type` 字段）。所以现在图里的失败分布**本来就是"primary root-cause"单标签分布**，不是 per-step 多标签分布。**Part 1 是必须做的（0 成本，纯文字澄清 + 从已有标签统计）；Part 2 是可选加分项（需要对已有轨迹重跑一次 LLM judge，只跑 judge、不跑 agent）。** 请勿谎称"我们已经报告了完整 per-step 分布"——论文当前没有。

---

## 0. Background (confirmed facts about the pipeline)

- **Attribution is single-label per trajectory.** The judge walks the Q1–Q7 decision tree (Fig. `fig:decision_tree`) in temporal order, applies it to the **earliest critical failure step**, stops at the first "Yes", and emits **one** `failure_type` ∈ {7 categories, `Others`, `Success`} (Appendix `app:4`, judge prompt lines ~2975–3029 and output schema ~3107–3115). "Focus on ROOT CAUSE at the earliest critical failure step — do not relabel the trajectory by symptoms of later steps" (line ~3028).
- **The taxonomy is *conceptually* orthogonal / non-mutually-exclusive** (main text §`subsec:failure_taxonomy`, line ~336: "a single failed trajectory may exhibit several categories across different steps"), but the **reported distribution** (`fig:failure_distribution`, and per-domain/model appendix figures) is the **single primary-root-cause distribution**. This is a deliberate design choice: attributing the root cause at the first irreversible divergence avoids double-counting downstream symptoms of the same failure.
- **The ordering that resolves overlaps** is the decision tree's causal, upstream-before-downstream, first-"Yes" ordering (lines ~2776, 2983; Fig. `fig:decision_tree`).

> **Do NOT claim** the paper currently reports a per-step multi-label distribution. It does not. State the pipeline is single-label by design; multi-label co-occurrence is an *optional* added analysis (Part 2).

---

## Part 1 — Single-Label Clarification (REQUIRED, zero-cost)

### Purpose
Answer "single vs. multi-label" definitively and make the paper internally consistent (right now §`subsec:failure_taxonomy` says "orthogonal / several categories" while the pipeline is single-label — a reviewer can read this as a contradiction).

### Task
1. From the **existing** 3100+ judge outputs, confirm and report: each trajectory has **exactly one** `failure_type`; count the distribution of `failure_type` (should reproduce `fig:failure_distribution` exactly — this is a sanity check).
2. Draft **two-level clarifying text** for the main text (see wording in §Deliverables) that separates:
   - **Trajectory level:** the 7 categories are *conceptually* orthogonal (a long trajectory can contain multiple failure steps), so they are **not** mutually exclusive as a descriptive space.
   - **Attribution level (what we report):** we assign a **single primary root-cause label** per trajectory via the earliest-critical-step + first-"Yes" decision-tree rule, so the reported distribution is a **single-label primary-cause distribution** (not double-counted).

### Deliverables
- `Exp4_singlelabel_distribution.csv`: `category, n_trajectories, share` (must equal `fig:failure_distribution`).
- A ready-to-paste sentence for §`subsec:failure_taxonomy`, e.g.:
  > "**Attribution is single-label at the trajectory level.** Although the seven categories are conceptually orthogonal and a long trajectory may contain several failure *steps*, our pipeline assigns one **primary root-cause** label per trajectory: the judge applies the Q1–Q7 decision tree (Fig.~\ref{fig:decision_tree}) to the *earliest critical failure step* and takes the first-``Yes'' branch, so upstream causes take precedence over downstream symptoms and no trajectory is double-counted. The distributions in Fig.~\ref{fig:failure_distribution} are therefore single-label primary-cause distributions."

---

## Part 2 — Optional Multi-Label Co-occurrence Analysis (VALUE-ADD; judge re-run only, NO agent re-run)

### Purpose
Directly address "the main distribution may not reflect multi-label failure composition" by *additionally* reporting which categories **co-occur** within the same trajectory — turning the reviewer's concern into a bonus result without changing the headline single-label distribution.

### Method (choose the cheapest viable path)
- **Path A (preferred, cheapest):** if the stored judge outputs already contain per-step reasoning / the extracted `steps` slices with per-step Q-labels, parse the **set** of categories touched across steps per trajectory. No new LLM calls.
- **Path B (if per-step labels were not stored):** run a **secondary judge pass** over the **existing** trajectories with a modified prompt that emits the **set** of all categories present across failure steps (multi-label), keeping the primary-root-cause label unchanged. **This re-runs only the LLM judge on existing trajectories — it does NOT re-run any agent and requires no new rollouts.** Use the same judge model (GPT-5) and record cost.

Then compute:
1. **Co-occurrence matrix** (7×7): for each pair of categories, the fraction of trajectories in which both appear.
2. **Cascade patterns:** most frequent ordered pairs/triples (e.g., False Assumption → Planning Error → History Error Accumulation, which the paper already hypothesizes qualitatively, line ~958 commented / §`sec:4`).
3. **Multi-label failure rate:** fraction of trajectories with ≥2 distinct categories; distribution of #categories per trajectory.
4. Confirm the **primary-cause distribution is stable**: the headline ordering (Planning / Memory / Catastrophic Forgetting dominate) should be unchanged whether you look at primary-cause or at multi-label presence.

### Deliverables
- `Exp4_cooccurrence_matrix.csv` + `Exp4_cooccurrence.pdf` (7×7 heatmap).
- `Exp4_cascade_patterns.csv`: top ordered pairs/triples with frequencies.
- `Exp4_multilabel_summary.csv`: `n_categories_per_traj` histogram + fraction ≥2.
- 2–3 sentences for the rebuttal.

### Expected result / framing
- Expect meaningful co-occurrence between **Planning Error, History Error Accumulation, and False Assumption** (cascades), and between **Memory Limitation and Catastrophic Forgetting** (both memory-related). This *supports* the paper's narrative that long-horizon failures compound.
- The dominant-category ordering should **persist** under the multi-label view — state this explicitly, so the single-label headline is shown to be robust, not an artifact of forcing one label.

---

## 3. Acceptance Criteria (self-check)

- [ ] Part 1 reproduces `fig:failure_distribution` exactly from existing judge outputs (sanity check).
- [ ] The rebuttal/paper text does NOT overclaim (no "we report full per-step distribution" unless Part 2 Path A/B is actually run).
- [ ] If Part 2 is run via Path B, it is explicitly noted that only the **judge** (not agents) was re-run on **existing** trajectories.
- [ ] Co-occurrence matrix and cascade counts have `n` and are reproducible (seed/model recorded).
- [ ] Report confirms whether the dominant-category ordering is stable across single-label vs. multi-label views.

---

## 4. Mapping to the rebuttal

Feeds: "**Attribution is single-label by design** (one primary root-cause per trajectory via the earliest-critical-step + first-``Yes'' decision tree, Fig.~\ref{fig:decision_tree}); we will state this explicitly in §3 and Fig.~\ref{fig:failure_distribution}. To address multi-label *composition*, we additionally report a category **co-occurrence** matrix and the most frequent failure cascades on the existing trajectories (judge-only pass, no agent re-runs); the dominant-category ordering is unchanged under this multi-label view."
