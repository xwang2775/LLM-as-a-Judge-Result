# Exp5 — Per-Domain Summary Table (# tasks, $H^*$, oracle action count, action budget, success rate + CI)

> **给执行 agent 的说明（中文）**：这是**纯分析实验，不需要重跑 agent**。目标是直接产出审稿人 Comment 里点名要的那张表："Add a table with, for each domain: number of tasks, estimated H, oracle/expert action count, action budget, and success rate with confidence intervals." 这张表大部分是**装配 Exp1 的产出**（$H^*$、oracle/expert action count、action budget 已在 Exp1 里算了）+ **本文档新增的一项：带置信区间的 success rate**（从已有的每设置 3 次 run 里 bootstrap 得到）。请先跑/读取 Exp1 的输出，再补 success-rate CI，最后拼成一张表。若 Exp1 尚未运行，可独立完成本表所需的最小子集，但要在报告里注明哪些数来自哪里。

---

## 0. Background & Motivation

**Reviewer comment (verbatim).** "Add a table with, for each domain: number of tasks, estimated H, oracle/expert action count, action budget, and success rate with confidence intervals."

The paper currently states action **budgets** (up to 150 Web / 50 OS / 31 DB / 34 Embodied steps; Appendix `app:2`) and plots success vs. the construction variable $s$ (`fig:horizon`), but does **not** put all of {#tasks, $H^*$, oracle action count, budget, success±CI} in one per-domain table. This table makes the "where"/scale of the benchmark auditable at a glance.

**No agent re-runs.** All quantities come from (i) the task specs/construction code, (ii) Exp1 outputs, and (iii) the existing 3100+ trajectories with their programmatic success labels over 3 runs.

---

## 1. Inputs (locate first)

- **Exp1 outputs** (if available): `B_Hstar_by_level.csv` (estimated $H^*$ per domain×level), `A_length_units.csv` (realized steps / tool calls = the oracle/expert action count proxy on successful trajectories), and the documented action budgets. Reuse these; do not recompute differently.
- **Task construction specs** — for `# tasks` per domain (and per level if you want the level-resolved version) and the **oracle/expert action count** (gold-solution / expert-demo length). Prefer gold-solution length as the "oracle/expert action count"; if unavailable, use median realized steps on *successful* trajectories as a proxy and label it as such.
- **Action budgets** — per-domain caps (Web 150 / OS 50 / DB 31 / Embodied 34; verify against `app:2`).
- **Trajectory dataset** — for `success` (programmatic/final-state label, NOT the LLM judge) and `run_id` (3 runs per setting), for the success-rate CIs.

> Confirm the success label is the domain's **objective** checker (WebArena/AgentBench checkers, DB execution/result match, Embodied final-state check), not the judge. Record this.

---

## 2. Task

Produce a table with **one row per domain** (and, in an appendix variant, one row per domain × $s$ level):

| Column | Source | Notes |
|---|---|---|
| `n_tasks` | construction specs | number of distinct base/extended tasks in the domain |
| `n_trajectories` | trajectory dataset | tasks × runs (report both n_tasks and n_traj) |
| `Hstar_range` or `Hstar_by_level` | Exp1 `B_*` | estimated intrinsic horizon (min–max across levels, or per level) |
| `oracle_action_count` | gold solution / expert demo | median [IQR]; label proxy if realized-steps used |
| `action_budget` | `app:2` | per-domain step cap |
| `success_rate` | trajectory dataset | overall and/or per level |
| `success_ci` | **NEW, this doc** | bootstrap 95% CI (see below) |

### Success-rate CI method (the new computation)
- **Unit of resampling = task** (resample tasks with replacement within the domain/level; average over the task's runs), B=10000, fixed seed recorded. This captures task-sampling uncertainty and respects that the 3 runs of a task are correlated.
- Additionally report a **Wilson score interval** on the raw success proportion as a simple cross-check.
- Report success rate at the domain level **and** per $s$ level (the per-level version is what pairs with `fig:horizon` and the collapse claim — see also Exp1 Part C for the breaking point).

---

## 3. Deliverables

- `Exp5_per_domain_summary.csv`: `domain, n_tasks, n_trajectories, Hstar_min, Hstar_max, oracle_action_count_med, oracle_action_count_iqr, action_budget, success_rate, success_ci_low, success_ci_high, success_ci_method`
- `Exp5_per_domain_level_summary.csv`: same but with an `s` column (per-level).
- `Exp5_summary_table.tex`: the compact per-domain table for the main text / `app:2`.
- 2–3 sentences summarizing the table for the rebuttal.

---

## 4. Expected result / framing

- One clean table that lets a reader see, per domain: how many tasks, the $H^*$ scale, the oracle action count, the budget headroom (budget ≫ oracle count, showing failures are not budget-starvation), and success±CI.
- **Budget vs. oracle gap matters:** showing `action_budget` ≫ `oracle_action_count` preempts "maybe they just ran out of steps" — failures happen well inside the budget. State this explicitly.
- Success CIs should be tight where n is large; where a level has few tasks, report wide CIs honestly.

---

## 5. Acceptance Criteria (self-check)

- [ ] Success label confirmed to be the objective programmatic checker, not the LLM judge.
- [ ] $H^*$ / oracle counts reuse Exp1 outputs (or, if computed here, use the identical method and say so).
- [ ] Every success rate has an `n` (tasks and trajectories) and a bootstrap 95% CI (B≥10000, seed recorded); Wilson interval also reported.
- [ ] Resampling unit = task (runs kept together), documented.
- [ ] `action_budget` values verified against `app:2` (Web 150 / OS 50 / DB 31 / Embodied 34).
- [ ] No agent re-runs; report states all inputs are existing artifacts.

---

## 6. Mapping to the rebuttal

Feeds: "We add a per-domain summary table (Table X) reporting, for each domain, the number of tasks/trajectories, estimated $H^*$, oracle action count, action budget, and success rate with bootstrap 95% CIs. Budgets substantially exceed oracle action counts, so failures are not caused by budget exhaustion. All values come from existing trajectories and task specs (no re-runs)."
