# Exp1 — Action-Level Horizon ($H^*$), Trajectory Length in Comparable Units, and Breaking-Point Quantification

> **给执行 agent 的说明（中文）**：这是为 EMNLP rebuttal 准备的**测量/分析实验**，用来回应审稿人关于"这真的是 long-horizon 吗？""长度是用词数而非动作数衡量的""where 分析只在 compositional depth $s$ 上、没有 action-level 依据"这一组意见（对应 W1 / W2 / Comment 1 / Comment 3）。
>
> **核心任务**：不重训、不重跑 agent（除非必要，见 §6 兜底）。基于**已有的 3100+ 条轨迹**和**任务构造脚本/规格**，为每个 domain × 每个 extension level $s$ 报告三样东西：(A) 每条轨迹的**真实长度**，用与近期文献可比的单位（**动作步数 / 工具调用数 / token / 词数**）;(B) **估计的 intrinsic horizon $H^*$**（最优策略所需的有效动作数）;(C) 把 success-vs-$s$ 曲线**重新表达在 action-level $H^*$ 轴上**，并**量化每个 model × domain 的 breaking point**（success 跌破阈值时的 $H^*$，带 bootstrap 置信区间）。
>
> **务必先做**：读真实数据、打印字段与单位，再对齐本文假设。若与本文假设不一致，以真实数据为准，并在报告里记录你实际用的字段、单位、以及 $H^*$ 的推导方式。严格按 §7 Acceptance Criteria 自查后再交付。

---

## 0. Background & Why This Experiment

**Paper.** *The Long-Horizon Task Mirage? Diagnosing Where and Why Agentic Systems Break* introduces `HORIZON`: two agent-independent task metrics — **Intrinsic Horizon $H^*$** (minimum effective actions an optimal policy needs) and **Compositional Depth $s$** (number of high-level subtasks) — plus controlled task extension (depth extension for OS/DB, breadth extension for Web/Embodied). It evaluates GPT-5-mini and Claude-4-Sonnet (+ GPT-5 on Embodied) on 3100+ trajectories across Web, OS, Database, Embodied.

**What the paper currently reports (verify these against the real repo before starting):**
- Success is plotted **only against the construction variable $s$** (Figure `fig:horizon`), not against an action-level scale.
- Trajectory *length* is described in **words**: "a single Web trajectory has a median length of ~6300 words (~25 pages)"; "OS trajectories average ~1800 words" (Appendix `app:4`, "Why 40 trajectories").
- Only **per-trajectory action budgets** are stated (up to **150 Web / 50 OS / 31 DB / 34 Embodied** steps, Appendix `app:2`, `subsec:Evaluation` / line ~1148). These are *budgets/caps*, **not** the realized $H^*$ per level.
- The paper asserts "$H^*(s)$ grows substantially faster than $s$" but **never reports the realized $H^*$ (or realized step counts) per extension level**.

**Reviewer concerns this addresses:**
- **W1 / Comment 1** — "length evidence is verbosity (words), not action count"; "report trajectory length in comparable units to recent literature (number of steps and tool calls, alongside words/tokens)"; "report per-level $H^*$ (or median actual optimal steps) per domain so the collapse point can be located on an action scale." The reviewer suspects the collapse sets in at tiny action counts (Web $H^*$ 3–8, Embodied 4 primitives) and thus looks like basic multi-subtask composition, not long-horizon.
- **W2** — "the *where* analysis is only aggregate accuracy-vs-$s$ curves with no validation, expressed in $s$; strengthen it to a validated, per-model×domain breaking point on the action-level $H^*$ scale."
- **Comment 3** — "no specific breaking-point value is reported; either quantify the breaking point per model×domain (the $H^*$ at which success crosses a set level, with confidence intervals) or adjust wording around *when*."

**Goal.** Convert "we only have $s$-axis curves and word counts" into "we report realized action-level length in literature-comparable units, we report estimated $H^*$ per level, and we quantify each model×domain breaking point on the $H^*$ axis with CIs." This directly repairs `fig:horizon`, Appendix `app:2` (`subsec:Evaluation`), and the "Breaking Points as Transition Regions" discussion.

---

## 1. Inputs (locate these first)

This folder is LaTeX-only; the data + construction code live in the sibling results/benchmark repo the authors used. Before anything, locate and print schemas for:

### 1.1 Trajectory dataset (3100+ trajectories)
Per-record fields (names may differ — **map them and record the mapping**):
- `trajectory_id`, `domain` ∈ {Web, OS, Database, Embodied}, `model` ∈ {GPT-5-mini, Claude-4-Sonnet, GPT-5 (Embodied only)}
- `s` (extension level) — **required**; if absent, recover from the task-id / task-set metadata.
- `run_id` (the paper reports mean±std over **3 runs** per horizon setting — keep runs separate for CIs).
- `success` (bool) — the domain's *programmatic / final-state* success label (Web: WebArena checker; OS: AgentBench checker; DB: execution/result match; Embodied: final object-configuration check). **This is objective ground truth, not the LLM judge** — confirm this.
- Raw trajectory (list of steps). Each step should expose the **agent action / tool call** and the **observation**. If only raw text exists, you must parse actions from it (see §2).

### 1.2 Task construction specs / scripts
The depth/breadth extension code or task specs. You need, per base task and per extension operation:
- The **base intrinsic horizon** $H^*_\text{base}$ (from WebArena reference solutions / oracle solvers / expert demos / the DB gold SQL decomposition / the embodied gold plan length).
- The **per-extension increment**: depth extension adds "1–2 permission checks" (OS) / "2–3 filtering operations" (DB) per level; breadth extension composes $k$ base tasks (so $s=k$) with coordination overhead $\epsilon$.
- Enough info to compute $H^*(s)$ analytically (preferred) or to read a gold/oracle solution length per task instance.

> If gold/oracle solutions exist per task instance, prefer **counting gold-solution action length** as $H^*$ (most defensible). Only fall back to the analytic $H^*_\text{base}+\Delta(s)$ formula if per-instance gold solutions are unavailable — and say which you used.

---

## 2. Part A — Realized Trajectory Length in Literature-Comparable Units

### Purpose
Answer W1/Comment 1 head-on: report length in **action steps, tool calls, tokens, and words** — not words alone — so the numbers are directly comparable to recent long-horizon benchmarks (which report steps/tool calls).

### Method
Define one **step** = one agent decision/turn that emits an action (parse from the trace's action/tool-call field; do **not** count observation-only lines). Define **tool calls** = number of environment/tool invocations (in Web = accessibility-tree actions like click/type/scroll; OS = shell commands; DB = SQL executions / sub-agent calls; Embodied = primitives in the emitted plan). If step and tool-call coincide for a domain, report them as equal and say so.

For **every domain × $s$ level** (and separately for successful vs failed trajectories, since failed ones often loop and inflate counts):
- `n_steps`: median [IQR] agent action steps per trajectory
- `n_tool_calls`: median [IQR] tool/environment calls
- `n_tokens`: median [IQR] tokens of the full trace (use a stated tokenizer, e.g. `tiktoken` `o200k_base`; state it)
- `n_words`: median [IQR] words (to reproduce/replace the current ~6300-word Web / ~1800-word OS figures)

### Deliverables
1. `A_length_units.csv`: `domain, s, split(success/fail/all), n_traj, steps_med, steps_iqr, toolcalls_med, toolcalls_iqr, tokens_med, tokens_iqr, words_med, words_iqr`
2. `A_length_table.tex`: a compact appendix table (domain × $s$, median steps / tool calls / tokens / words).
3. One sentence-ready summary of realized action counts per domain to drop into `app:2`.

### Desired / expected result
- The Web ~6300-word median maps to a **concrete step/tool-call count** (this is the key number to surface). Report it honestly even if smaller than expected.
- If realized step counts at the breaking level are modest (e.g., Web low-tens), **do not hide it** — instead pair it with $H^*$ (Part B) and the argument that long-horizon is defined by *long-range dependency across subgoals*, not raw step count (see the rebuttal note in §8). The reviewer explicitly suspects small counts; an honest number + the dependency argument is stronger than an evasive one.

---

## 3. Part B — Estimated Intrinsic Horizon $H^*$ per Level

### Purpose
Give the **action-level** $H^*(s)$ the paper claims exists but never tabulates, so the $x$-axis of the "where" analysis can be an action scale (W1/W2/Comment 1/Comment 3).

### Method
For each domain, produce $H^*(s)$ for every level $s$ used in the experiments:
- **OS, DB (depth extension):** $H^*(s) = H^*_\text{base} + \Delta(s)$. Prefer per-instance gold-solution length; else use the construction increments (OS: +1–2 checks/level; DB: +2–3 filtering ops/level) and state the assumed increment.
- **Web, Embodied (breadth extension):** $H^*(s) = \sum_{i=1}^{s} H^*_i + \epsilon(s)$, summing the base intrinsic horizons of the composed subtasks; report $\epsilon$ (coordination overhead) separately if you can estimate it, else set $\epsilon=0$ and label $H^*$ as a **lower bound**.

Report **both** the estimated $H^*$ **and** the median realized optimal steps (from Part A on *successful* trajectories, which upper-bounds $H^*$ per instance) so the reader sees theory vs. practice.

### Deliverables
1. `B_Hstar_by_level.csv`: `domain, s, Hstar_estimate, Hstar_method(gold|analytic), Hstar_lb_ub, realized_steps_success_med`
2. `B_Hstar_table.tex`: domain × $s$ → $H^*$ table for `app:2`.
3. A short note on the **$s \to H^*$ mapping per domain** (this is exactly what Comment 2 asks to be made explicit).

### Desired / expected result
- Demonstrate that a single unit of $s$ expands to **several** $H^*$ actions (supports the paper's "$H^*$ grows faster than $s$" claim with actual numbers).
- Show that by the breaking level, $H^*$ reaches the **tens** in Web/OS/DB (and give the honest Embodied number). This reframes Figure `fig:horizon` on an action scale.

---

## 4. Part C — Per-Model×Domain Breaking Point on the $H^*$ Axis (with CIs)

### Purpose
Answer W2 + Comment 3: produce a **quantified, validated** breaking point per model×domain on an **action scale**, not a qualitative region in $s$.

### Method
1. Re-express each success-vs-$s$ curve as **success-vs-$H^*$** using the mapping from Part B.
2. Define the breaking point $H^*_\tau$ = the $H^*$ at which the fitted success rate crosses a threshold $\tau$ (report $\tau \in \{0.75, 0.5, 0.25\}$; make $\tau=0.5$ the headline). Interpolate between adjacent levels (linear or logistic fit — state which).
3. **Bootstrap 95% CI** on $H^*_\tau$: resample **tasks within each level** (and/or resample the 3 runs) with replacement, B=10000, fixed seed; recompute $H^*_\tau$ each resample; report 2.5/97.5 percentiles.
4. Do this for **each (model, domain)** cell, so the reviewer's requested "per-model×domain breaking point" table exists.

> **On "validation" of the *where* side (W2):** unlike the "why" side (which needs κ because attribution is subjective), the "where" side rests on **objective, programmatic success oracles** (WebArena/AgentBench checkers, DB execution match, Embodied final-state check). So the appropriate rigor for "where" is **statistical** (CIs from repeated runs + bootstrap over tasks), which this part supplies. Make this explicit in both the paper and rebuttal. If Embodied success is human-checked, note it is a deterministic final-state criterion (not subjective judgment).

### Deliverables
1. `C_breaking_points.csv`: `model, domain, tau, s_at_tau, Hstar_at_tau, Hstar_ci_low, Hstar_ci_high, n_tasks_per_level`
2. `C_breaking_point_table.tex`: model × domain → $H^*_{\tau=0.5}$ [CI] for the main text / `app:2`.
3. `C_success_vs_Hstar.pdf`: the `fig:horizon` panels re-plotted with $H^*$ on the $x$-axis and shaded breaking regions + CIs.

### Desired / expected result
- Each cell yields a concrete "success crosses 50% at $H^* \approx X$ [CI]" statement.
- Cross-domain differences persist on the $H^*$ axis (e.g., Web breaks at smaller $H^*$ than OS/DB), quantifying the paper's "different domains break at different horizons" claim.
- If breaking points have wide CIs at some cells (few levels/tasks), report honestly and frame as "transition region [CI]" consistent with the paper's existing language.

---

## 5. Output Bundle & Report
Create `exp1_analysis/` containing:
- All CSVs (`A_*.csv`, `B_*.csv`, `C_*.csv`), all `.tex` snippets, `C_success_vs_Hstar.pdf`.
- One reproducible, seeded script `exp1_analysis.py` with `--data-path` and `--construction-spec-path` args, and a `README.md` documenting: data files used, field mapping, how "step"/"tool call" were parsed per domain, tokenizer, $H^*$ derivation (gold vs analytic), seed.
- `Exp1_findings.md` (≤1.5 pages):
  1. Realized length per domain in steps / tool calls / tokens / words (the table).
  2. $H^*(s)$ per domain and the $s\to H^*$ mapping.
  3. Per-model×domain breaking point $H^*_{0.5}$ [CI].
  4. Any result that **weakens** a paper claim (e.g., if a domain's realized $H^*$ is genuinely tiny) — flag prominently so the rebuttal wording can be adjusted.

---

## 6. Contingency (only if data insufficient)
- If per-instance gold/oracle solutions do not exist and construction increments are ambiguous, compute $H^*$ **two ways** (analytic + realized-successful-steps proxy) and report both as bounds — do **not** invent a single number.
- Re-running agents is **not** required for any part. Only if the trajectory dumps lack parseable actions for a domain should you flag it to the authors (do not silently guess step counts from prose).

---

## 7. Acceptance Criteria (self-check before delivering)
- [ ] Reproduced the **existing** success-vs-$s$ numbers from the located data before re-expressing on $H^*$ (sanity check the pipeline). If you cannot reproduce Figure `fig:horizon`, stop and report.
- [ ] "Step" and "tool call" definitions are explicit **per domain**, counted from action/tool fields (not observation lines), and documented.
- [ ] Length reported in **all four units** (steps, tool calls, tokens, words); the ~6300-word Web / ~1800-word OS figures are reproduced (or corrected) and mapped to step counts.
- [ ] $H^*(s)$ reported per domain×level with the derivation method stated (gold vs analytic) and marked as estimate/lower-bound where appropriate.
- [ ] Every breaking point has an `n`, a $\tau$, and a bootstrap 95% CI; B≥10000; seed recorded.
- [ ] `Exp1_findings.md` states clearly whether any result weakens a claim (honesty over optimism).
- [ ] Everything regenerable via `python exp1_analysis.py --data-path <...> --construction-spec-path <...>`.

---

## 8. Mapping to the rebuttal (for reference)
Feeds these rebuttal points:
- **W1/C1:** "We now report realized trajectory length in steps, tool calls, and tokens (Table X), and estimated $H^*$ per level (Table Y). At the breaking level, Web/OS/DB reach $H^*$ in the tens; the ~6300-word Web median corresponds to ~N steps / ~M tool calls."
- **W2/C3:** "We now quantify a per-model×domain breaking point $H^*_{0.5}$ with bootstrap CIs (Table Z) and re-plot the where-analysis on the action-level $H^*$ axis (Fig. C). The where-side rests on objective programmatic success oracles, so we validate it statistically (repeated runs + bootstrap) rather than via κ."
- **Long-horizon vs complex-reasoning:** pair the action counts with the dependency argument (long-range constraint retention across subgoals), not raw step magnitude.
