# 07. Four Data Splits and the S / R Scores

Source: PPTX page 7 of 17.

| Order | Short-clip split size | Purpose | Decision signal |
|---|---|---|---|
| Train | 4 cases | Reflection and parent/child comparison; only these trajectories are exposed | S_clip |
| Validation | 2 cases | Candidate ranking, Pareto frontier, and best selection | S_clip |
| Promotion | 6 cases | Independent reevaluation and promotion gate | R_clip and quality axes |
| Test | 4 cases | Final report; no feedback to search or promotion | R_clip |

**Metric names specify both role and evaluation unit.** Use `R_clip` / `S_clip` for one generated clip, and `R_project` / `S_project` for a complete multi-clip project. R denotes underlying reward/reporting and S denotes the search objective; neither name identifies when a score first becomes available. The counts above describe short-clip tasks, not longform projects.

| Metric | Unit | Meaning |
| --- | --- | --- |
| `R_clip` | One task/segment, harness and seed | Underlying weighted clip reward, measured during search and independent evaluation |
| `S_clip` | The same clip evaluation | Search objective derived from that case's R_clip and enabled axis-regression penalty |
| `S_project` | One complete project, harness and seed | Search objective built from project metrics, including mean(R_clip) |
| `R_project` | The same complete-project evaluation | Reward/report score with additional state/payoff terms; used for independent project promotion |

### Short clips

`R_clip = 0.60 video + 0.25 story + 0.15 editing`

`S_clip = R_clip - axis-regression penalty`

For each Train/Validation evaluation, measure the current clip's `R_clip`, then derive `S_clip` from that same case. The penalty uses enabled quality-axis regressions against the base on the same case; it does not use Promotion/Test results. With protection disabled or no axis regression, `S_clip = R_clip`.

| Decision stage | Evaluation data | Score used |
| --- | --- | --- |
| Search and selection | Train comparisons and Validation ranking | Compute current-case R_clip, derive S_clip, and use S_clip for decisions |
| Independent promotion | Fresh Base/winner evaluations on held-out Promotion tasks | Use their newly measured R_clip and quality-axis safeguards |
| Final reporting | Independent Test tasks | Report their R_clip and axes; no feedback to selection |

Thus, using a search score before promotion describes the **decision order**. It does not mean search needs a future promotion reward. Story and editing are text/planning proxies, not direct measurements of finished-film quality.

### Long-form projects

`C = mean(R_clip)` across the project's segments

`S_project = 0.70 C + 0.30 H`

`R_project = 0.40 C + 0.20 F + 0.20 P + 0.20 H`

F is state confirmation, P is payoff, and H is visual handoff: similarity between adjacent last/first frames. Both project scores use the current project's metrics. The shared component C is the mean **raw clip reward**, not R_project or the penalized clip search score S_clip. The current longform implementation does not carry the short-clip axis-regression penalty into the project formula.

Train/Validation use `S_project` for project search. Independent Promotion uses newly evaluated held-out `R_project` and safeguards; Test reports its own `R_project`. R_project can also be recorded on search projects as a diagnostic without driving selection.

These project formulas assume all metrics are present and loop penalty is zero; missing metrics follow the frozen policy. A higher search score does not guarantee a higher reward/report score or better output in every case. Clip-level and project-level values have different definitions and cannot be compared directly. The demonstrated runs use these scores for harness search/evaluation, not direct model-weight updates.

For concrete examples, see the [single-clip evaluation](subdocs_examples/08-script-rewrite-prompt-lifecycle.md#10-evaluation-and-what-the-example-demonstrates) and the [five-clip reward aggregation and project evaluation](subdocs_examples/11-town-fair-longform-production.md#7-clip-rewards-and-project-evaluation). The latter maps recorded fields to clip and project metrics; missing historical values remain marked as DGX placeholders.

### Short-clip run `101214_1strun`

The original chart reports mean R_clip × 100:

| Split | Base | Optimized | R_clip gain on the 0–1 scale |
|---|---:|---:|---:|
| Validation | 65.88 | 67.49 | +0.0161 |
| Promotion | 61.07 | 65.28 | +0.0421 |
| Test | 59.05 | 63.93 | +0.0488 |

These chart values are preserved from the embedded chart data. [Page 8](08-short-clip-results.md) shows one Train task's R_clip/video scores; [page 12](12-town-fair-lineage.md) shows mean S_project/R_project over four Validation projects, rather than town-fair alone.

### Source case files

Full inputs: [single-clip JSONL](../benchmarks/tasks_gepa_v1.jsonl) and [all five-clip YAML projects](../benchmarks/README.md#five-clip-inputs), with split assignments and [SHA-256 checksums](../benchmarks/checksums.json).

---

[Contents](README.md) · [Previous](06-gepa-loops.md) · [Next](08-short-clip-results.md)
