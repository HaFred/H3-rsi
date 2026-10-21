# 07. Four Data Splits and the S / R Scores

Source: PPTX page 7 of 17.

| Order | Short-clip split size | Purpose | Decision signal |
|---|---|---|---|
| Train | 4 cases | Reflection and parent/child comparison; only these trajectories are exposed | S |
| Validation | 2 cases | Candidate ranking, Pareto frontier, and best selection | S |
| Promotion | 6 cases | Independent reevaluation and promotion gate | R and quality axes |
| Test | 4 cases | Final report; no feedback to search or promotion | R |

**R is the underlying reward/report score; S is the objective used for search.** R is computed on search cases too, not reserved for a later reporting stage. "Use S, then R" describes the order of selection and independent promotion decisions, not a dependency on future promotion scores. On this page, `R` / `S` denote clip-level scores and `R_long` / `S_long` denote complete-project scores (called `R_clip` / `S_clip` and `R_project` / `S_project` elsewhere). The counts above describe the short-clip dataset; they do not describe the long-form validation set.

### Short clips

`R = 0.60 video + 0.25 story + 0.15 editing`

`S = R - axis-regression penalty`

For each Train/Validation evaluation, the evaluator generates a clip and computes its `R`; GEPA uses that same case's penalized `S` to compare and rank candidates. The penalty measures enabled quality-axis regressions against the base on the same case. It does not use Promotion/Test results. With protection disabled or no axis regression, `S = R`.

| Decision stage | Evaluation data | Score used |
| --- | --- | --- |
| Search and selection | Train comparisons and Validation ranking | Compute current-case `R`, then derive `S`; use S for search decisions. |
| Independent promotion | Fresh Base/winner evaluations on held-out Promotion tasks | Use their newly measured `R` and quality-axis safeguards to accept/reject the selected candidate. |
| Final reporting | Independent Test tasks | Report their `R` and axes; results do not feed back into selection. |

Story and editing are text/planning proxies, rather than direct measurements of finished-film quality.

### Long-form projects

`S_long = 0.70 mean(R) + 0.30 H`

`R_long = 0.40 mean(R) + 0.20 F + 0.20 P + 0.20 H`

Here `mean(R)` averages the clip reward defined above across the project's segments. Since `S = R - axis-regression penalty`, the same component can also be written `mean(R) = mean(S + axis-regression penalty)`. The current long-form implementation aggregates the raw clip rewards; it does not carry the short-clip search penalty into the project score.

Here F is state confirmation, P is payoff, and H is visual handoff: similarity between a segment's last frame and the following segment's first frame. Both project scores are computed from the current project's metrics, with different weights; neither needs a future evaluation result. The shared `mean(R)` component is clip reward, distinct from `R_long`. These formulas assume all metrics are present and loop penalty is zero. Missing metrics are renormalized according to the frozen policy.

Train/Validation use `S_long` for search. Independent Promotion uses freshly evaluated `R_long` and safeguards; Test reports `R_long`. R_long can also be recorded on Train/Validation for diagnostics without becoming the selection objective.

An increase in S_long does not guarantee an increase in R_long; the same caution applies to short-clip S/R when penalties differ. Clip- and project-level scores have different definitions and should not be compared directly. These are frozen evaluation objectives; they do not directly train model weights in the demonstrated runs.

### Short-clip run `101214_1strun`

The original chart reports mean R × 100:

| Split | Base | Optimized | R gain on the 0–1 scale |
|---|---:|---:|---:|
| Validation | 65.88 | 67.49 | +0.0161 |
| Promotion | 61.07 | 65.28 | +0.0421 |
| Test | 59.05 | 63.93 | +0.0488 |

These chart values are preserved from the embedded chart data. [Page 8](08-short-clip-results.md) shows one Train task's R/video scores; [page 12](12-town-fair-lineage.md) shows mean S_long/R_long over four Validation projects, rather than town-fair alone.

### Source case files

Full inputs: [single-clip JSONL](../benchmarks/tasks_gepa_v1.jsonl) and [all five-clip YAML projects](../benchmarks/README.md#five-clip-inputs), with split assignments and [SHA-256 checksums](../benchmarks/checksums.json).

---

[Contents](README.md) · [Previous](06-gepa-loops.md) · [Next](08-short-clip-results.md)
