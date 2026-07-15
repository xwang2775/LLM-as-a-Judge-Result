# W5 Rebuttal Experiment Plan — Judge Reliability & Uncertainty Quantification

> **给执行 agent 的说明（中文）**：这是为 EMNLP rebuttal 准备的**轻量补充分析**，目标是回应审稿人 W5。**不需要重新跑任何 agent，也不需要调用任何 LLM。** 全部基于已有的 (1) 3100+ 条 judge 标注的失败轨迹，和 (2) 40 条人工标注的校准集。产出三样东西：带 95% 置信区间的失败分布图、分 domain 的 human–judge κ、分轨迹长度的 human–judge κ。请严格按"Acceptance Criteria"自查后再交付。若数据 schema 与本文假设不同，**先读真实数据、打印字段，再对齐**，并在最终报告里记录你实际用的字段与单位。

---

## 0. Background & Motivation

**Paper.** *Why Do Long-Horizon Agents Fail?* introduces HORIZON, a cross-domain benchmark (Web, OS, Embodied, Database) with a 7-category failure taxonomy, plus a trajectory-grounded **LLM-as-a-Judge** pipeline that labels 3100+ failure trajectories. Human validation covers **40 trajectories** (human–judge Cohen's $\kappa = 0.84$; inter-human $\kappa = 0.61$).

**Reviewer W5 (verbatim concern).** Only 40 trajectories are human-validated; the other 3060+ are labeled entirely by the LLM judge. The reviewer asks for (a) **confidence intervals** on per-category failure rates, and (b) evidence on **whether judge agreement degrades on out-of-distribution domains or longer trajectories**, where the small human-calibration set may not be representative.

**Goal of this work.** Produce three analyses, using **existing data only**, that convert this concern from "unaddressed" into "quantified and addressed":

1. **A1 — Bootstrap confidence intervals** on every per-category failure-rate figure.
2. **A2 — Per-domain human–judge agreement** (reliability is stable across domains, not just in aggregate).
3. **A3 — Per-trajectory-length human–judge agreement** (reliability does not collapse on the longest trajectories).

These directly feed Appendix `app:4` (Experiment 2, LLM-as-a-Judge) and the main-text failure-distribution figure (`fig:failure_distribution`).

---

## 1. Inputs (locate these first)

The LaTeX-only folder here does **not** contain the data. Before doing anything, locate the real artifacts (likely in the sibling project / results repo the authors used). You will need:

### 1.1 Judge-labeled trajectories (for A1)
The full set of **3100+ failure trajectories** with LLM-judge labels. Expected per-record fields (names may differ — map them):
- `trajectory_id` (unique)
- `domain` ∈ {Web, OS, Embodied, Database}
- `model` ∈ {GPT-5-mini / GPT family, Claude-4-Sonnet / Claude family}
- `horizon_level` (`s`) if available
- **Per-step labels** OR a **trajectory-level label list**: the failure category(ies) assigned. Categories: `Environment Error`, `Instruction Error`, `Catastrophic Forgetting`, `Memory Limitation`, `False Assumption`, `Planning Error`, `History Error Accumulation`, (`Others`).
- Raw trajectory text or a `length` field (token/word/char count). If absent, compute length from the raw trace.

### 1.2 Human-validation set (for A2, A3)
The **40-trajectory human-labeled set** used for the $\kappa=0.84$ human–judge study (one co-author vs. judge). Expected fields:
- `trajectory_id`, `domain`, `length` (or raw text to compute it)
- `human_label` (7-category, and sub-category if available)
- `judge_label` (aligned to the same trajectory/step unit as the human)

> **Also locate the existing κ-computation script** (whatever produced $\kappa=0.84$). **Match its unit of analysis exactly** (per-step vs. per-trajectory-primary-label, label set = 7 vs. 8 with Others) so all new κ numbers are directly comparable to the reported 0.84. Record the unit in the report.

**If the human-validation set is a *different* 40 than the inter-human set:** that's expected (the paper says the H–J study uses a separate 40-sample and a third annotator). Use the **H–J** set for A2/A3.

---

## 2. A1 — Bootstrap Confidence Intervals on Failure-Rate Distributions

### Purpose
Quantify the sampling uncertainty of the judge-derived per-category failure rates, so the reader knows the bars in the failure-distribution figures are not point estimates presented as exact.

### Statistic
For each **grouping** below, and each of the 7 (or 8 incl. Others) categories, compute the **category share** = the fraction of attributed failures assigned to that category (the exact quantity currently plotted in the paper's failure-distribution bars — **verify against the existing plotting code and reproduce the same numerator/denominator**).

Groupings (match the existing figures):
- **Overall** (main-text `fig:failure_distribution`)
- **By domain** (`appendix_by_domain.pdf`, 4 panels)
- **By model** (`appendix_by_model.pdf`, 2 panels)
- **By domain × model** (`appendix_by_domain_model.pdf`, 8 panels)

### Method (bootstrap over trajectories)
```
B = 10000                      # bootstrap resamples
unit = trajectory              # resample whole trajectories WITH REPLACEMENT
                               # (preserves within-trajectory correlation of step labels)
for each group G:
    obs = list of trajectories in G
    for b in 1..B:
        sample |obs| trajectories from obs WITH REPLACEMENT
        recompute each category's share on the resample
    for each category c:
        point = share on the original (unresampled) data
        CI_low, CI_high = 2.5th, 97.5th percentiles of the B resampled shares
```
- **Resample the trajectory, not the step**, because steps within a trajectory are correlated.
- Set a fixed random seed (e.g., 12345) for reproducibility; record it.
- Report the sample size `n` (number of trajectories) for every group.

### Deliverables
1. Updated figures with **95% CI error bars** on every bar:
   - `failure_distribution_ci.pdf` (main text)
   - `appendix_by_domain_ci.pdf`, `appendix_by_model_ci.pdf`, `appendix_by_domain_model_ci.pdf`
   - Keep visual style identical to the originals; just add error bars.
2. A machine-readable table `A1_failure_rate_ci.csv` with columns:
   `group_type, group_name, category, n_trajectories, point_estimate, ci_low, ci_high`
3. A LaTeX table snippet `A1_ci_table.tex` (overall + per-domain) for the appendix.

### Desired / expected result
- CIs should be **reasonably tight** for the dominant categories in large groups (Planning Error, Memory Limitation, Catastrophic Forgetting), because `n` per group is large (hundreds).
- **The qualitative claim must survive:** the three dominant categories should remain clearly separated from the rest **with non-overlapping or barely-overlapping CIs** against the minor categories. If any dominant-vs-minor ordering becomes ambiguous once CIs are added, flag it prominently — this changes what we can claim in Sec. 5 ("three categories consistently dominate").

---

## 3. A2 — Per-Domain Human–Judge Agreement

### Purpose
Show reliability is stable across all four domains, addressing "the calibration set may not be representative" / "OOD domains." The paper currently reports only the aggregate $\kappa=0.84$.

### Method
- On the 40-trajectory H–J set, split by `domain`.
- For each domain, compute **Cohen's $\kappa$** (human vs. judge) at the **same unit** as the reported 0.84.
- Also compute **raw agreement (%)** per domain (robust when a domain's label set is small/degenerate — κ can be undefined or unstable when one label dominates).
- Add **bootstrap 95% CI on κ** per domain (resample trajectories within the domain, B=10000). Expect wide CIs because per-domain `n≈10`; **report them honestly** rather than presenting per-domain κ as precise.

### Deliverables
1. `A2_per_domain_agreement.csv`: `domain, n, cohen_kappa, kappa_ci_low, kappa_ci_high, raw_agreement, n_categories_present`
2. LaTeX snippet `A2_per_domain_table.tex` for Appendix `app:4`.

### Desired / expected result
- Per-domain κ (or raw agreement) should stay in the **substantial range** and be broadly consistent across domains — no domain should be an outlier with near-chance agreement.
- **Frame the caveat proactively:** per-domain `n≈10` means these are *indicative*, not definitive; the aggregate $\kappa=0.84$ remains the headline. If one domain is notably lower, report it and note it as a target for the additional targeted human batch (see §5).

---

## 4. A3 — Per-Trajectory-Length Human–Judge Agreement

### Purpose
Directly test the reviewer's "longer trajectories" concern: does the judge get less reliable as trajectories get longer?

### Method
- Compute `length` for each of the 40 H–J trajectories (tokens preferred; else words or chars — **use the same length metric consistently**, and state which).
- Bin into **terciles** (short / medium / long) by length. If 40 splits unevenly, use `≈13/13/14`. Alternatively bin by `horizon_level s` if that is the more natural "length" axis — **do both if data allows**, and report whichever the reviewer's concern maps to (raw trajectory length is the literal reading).
- Per bin: **Cohen's κ** (+ bootstrap 95% CI, B=10000) and **raw agreement (%)**, at the reported unit.
- Report the length range covered by each bin (min/median/max) so readers see the "long" bin really is long (cross-reference the ~6,300-word median Web trajectory noted in `app:4`).

### Deliverables
1. `A3_per_length_agreement.csv`: `bin, length_metric, length_min, length_median, length_max, n, cohen_kappa, kappa_ci_low, kappa_ci_high, raw_agreement`
2. LaTeX snippet `A3_per_length_table.tex`.
3. Optional small plot `A3_kappa_vs_length.pdf` (κ with CI vs. length bin).

### Desired / expected result
- κ should **not drop sharply** from short → long bins. A flat or gently-varying trend supports "the judge remains reliable on long trajectories."
- **If κ does drop meaningfully on the long bin**, report it honestly and pivot the rebuttal to: (i) the drop's magnitude and CI overlap, and (ii) the commitment in §5 to add a targeted long-trajectory human batch. Do **not** hide a degradation — an honest, quantified answer scores better than an overclaim a reviewer can puncture.

---

## 5. Contingency: targeted long-trajectory / per-domain human batch (only if needed)

Only if A2 or A3 shows a bin/domain that is under-populated (e.g., `n<8`) **or** shows a concerning agreement drop:
- Draw a **small additional stratified human-validation batch** (e.g., 15–20 trajectories) sampled from the **longest** trajectories and/or the weakest domain, using the **same annotation protocol and rubric** as the original 40.
- Re-run A2/A3 with the augmented set; report both original and augmented numbers.
- This is a human-annotation task (author time), not compute — flag to the authors rather than executing autonomously.

---

## 6. Output Bundle & Report

Create a directory `w5_analysis/` containing:
- All CSVs (`A1_*.csv`, `A2_*.csv`, `A3_*.csv`)
- All figures (`*_ci.pdf`, `A3_kappa_vs_length.pdf`)
- All LaTeX snippets (`*.tex`)
- A single analysis script `w5_analysis.py` (reproducible, seeded, with a `--data-path` arg) and a short `README.md` documenting: data files used, field mapping, unit of analysis, seed, and how to regenerate everything.
- `W5_findings.md`: a 1-page plain-language summary structured as:
  1. What each analysis shows (with the key numbers).
  2. Whether the paper's dominant-category claim survives CIs (yes/no + evidence).
  3. Whether reliability is stable across domains and lengths (yes/no + numbers).
  4. Any red flags requiring a rebuttal wording change.

---

## 7. Acceptance Criteria (self-check before delivering)

- [ ] Reproduced the **existing** overall failure-rate numbers and $\kappa=0.84$ from the located data **before** adding CIs/splits (sanity check that the pipeline is wired correctly). If you cannot reproduce 0.84 (±0.02), stop and report the discrepancy — do not proceed on mismatched data.
- [ ] Bootstrap resamples **trajectories**, not steps; B ≥ 10000; fixed seed recorded.
- [ ] Every reported rate/κ has an `n` and a 95% CI.
- [ ] Unit of analysis (per-step vs. per-trajectory; 7 vs. 8 labels) is identical across A1/A2/A3 and matches the paper's reported numbers; documented explicitly.
- [ ] Figures visually match the originals plus error bars; no restyling.
- [ ] `W5_findings.md` states clearly whether any result **weakens** a paper claim (honesty over optimism).
- [ ] Everything regenerable via `python w5_analysis.py --data-path <...>`.

---

## 8. Mapping to the rebuttal (for reference)

The results plug into this rebuttal paragraph (already drafted):

> (1) **Confidence intervals:** bootstrap 95% CIs as error bars on every per-category failure-rate figure (`app:4`). (2) **Reliability across domains:** per-domain human–judge κ (the 40-set is stratified across 4 domains × 7 categories). (3) **Reliability on longer trajectories:** human–judge agreement stratified by trajectory length, testing directly whether the judge degrades on the longest trajectories; if a bin is under-populated, add a small targeted human batch. None of this requires re-running any agents.

**Do not overclaim in the rebuttal beyond what the numbers support.** Update the drafted paragraph if A1–A3 reveal anything unexpected.
