# 07. Four Data Splits and the S / R Scores

Source: PPTX page 7 of 17.

| Order | Short-clip split size | Purpose | Decision signal |
|---|---|---|---|
| Train | 4 cases | Reflection and parent/child comparison; only these trajectories are exposed | S |
| Validation | 2 cases | Candidate ranking, Pareto frontier, and best selection | S |
| Promotion | 6 cases | Independent reevaluation and promotion gate | R and quality axes |
| Test | 4 cases | Final report; no feedback to search or promotion | R |

S means **Search score**. R means **Reporting score**, also used in promotion evaluation. The counts above describe the short-clip dataset; they do not describe the long-form validation set.

### Short clips

`R = 0.60 × video + 0.25 × story + 0.15 × editing`

`S = R − axis-regression penalty`

Story and editing are text/planning proxies, rather than direct measurements of finished-film quality.

### Long-form projects

`S = 0.70 × segment quality + 0.30 × visual handoff`

`R = 0.40 × segment quality + 0.20 × state confirmation + 0.20 × payoff + 0.20 × visual handoff`

Segment quality is the mean per-segment R. Visual handoff measures similarity between a segment's last frame and the following segment's first frame. These formulas assume all metrics are present and loop penalty is zero. Missing metrics are renormalized according to the frozen policy.

An increase in S does not guarantee an increase in R. Short-clip and long-form scores have different definitions and should not be compared directly. Both are frozen evaluation proxies; they do not directly train model weights.

### Short-clip run `101214_1strun`

The original chart reports mean R × 100:

| Split | Base | Optimized | R gain on the 0–1 scale |
|---|---:|---:|---:|
| Validation | 65.88 | 67.49 | +0.0161 |
| Promotion | 61.07 | 65.28 | +0.0421 |
| Test | 59.05 | 63.93 | +0.0488 |

These chart values are preserved from the embedded chart data. [Page 8](08-short-clip-results.md) shows one Train task's R/video scores; [page 12](12-town-fair-lineage.md) shows mean S/R over four Validation projects, rather than town-fair alone.

### Source case files

Full inputs: [single-clip JSONL](../benchmarks/tasks_gepa_v1.jsonl) and [all five-clip YAML projects](../benchmarks/README.md#five-clip-inputs), with split assignments and [SHA-256 checksums](../benchmarks/checksums.json).

---

[Contents](README.md) · [Previous](06-gepa-loops.md) · [Next](08-short-clip-results.md)
