# 07. Four Data Splits and the S / R Scores

Source: PPTX page 7 of 17.

| Order | Short-clip split size | Purpose | Decision signal |
|---|---|---|---|
| Train | 4 cases | Reflection and parent/child comparison; only these trajectories are exposed | S_clip |
| Validation | 2 cases | Candidate ranking, Pareto frontier, and best selection | S_clip |
| Promotion | 6 cases | Independent reevaluation and promotion gate | R_clip and quality axes |
| Test | 4 cases | Final report; no feedback to search or promotion | R_clip |

**Metric names specify both role and evaluation unit.** Use `R_clip` / `S_clip` for one generated clip, and `R_project` / `S_project` for a complete multi-clip project. R denotes underlying reward/reporting and S denotes the search objective; neither name identifies when a score first becomes available. The counts above describe short-clip tasks, not longform projects.

| Metric | Unit | Meaning | Who produces it? | Connection to [page 4](04-service-topology.md) |
| --- | --- | --- | --- | --- |
| `R_clip` | One task/segment, harness and seed | Underlying weighted clip reward, measured during search and independent evaluation | Local `RewardBuilder` combines video from UnifiedReward, story from Jev, and editing from a local storyboard heuristic. | Visual judge `:8090`, Decisions `:8085`, plus local code. |
| `S_clip` | The same clip evaluation | Search objective derived from that case's R_clip and enabled axis-regression penalty | Local `VideoGEPAAdapter` subtracts the axis-regression penalty from `R_clip`. | No additional model call. |
| `S_project` | One complete project, harness and seed | Search objective built from project metrics, including mean(R_clip) | Local project scorer combines mean `R_clip` (C) and locally calculated boundary-frame similarity (H). | Inherits the judges used for `R_clip`; H requires no model. |
| `R_project` | The same complete-project evaluation | Reward/report score with additional state/payoff terms; used for independent project promotion | Local project scorer combines C, H, confirmed-fact rate (F), and payoff rate (P). | Adds evidence/payoff judgments from Visual judge `:8090`. |

### Short clips

`R_clip` = 0.60 [video](09-short-clip-videos.md#metric-definitions) + 0.25 `story` + 0.15 `editing`

The evaluation instructions below are fixed during the run, separate from the production agents' `prompts.*` system messages. RSI rewrites the production harness; the evaluators score its resulting artifacts.

| Metric | Judge / implementation | Evaluation prompt or rule | Inputs |
| --- | --- | --- | --- |
| **Video** | UnifiedReward, `:8090` | Fixed `UNIFIED_REWARD_PROMPT`: rate alignment with caption, visual coherence and style, each 1-5. Sent as a user message, with no separate system message supplied by this scorer. | Sampled video frame + original task prompt as caption |
| **Story: script quality (60%)** | Jev, `:8085`, `Score` request | 评估该剧本的剧情完整度、冲突强度与节奏。 ("Evaluate plot completeness, conflict strength and pacing.") Five levels: very weak to very strong. | Generated script |
| **Story: key-event consistency (40%)** | Jev, `:8085`, `Noul` request | 生成描述是否完整兑现了剧本的关键事件？ ("Does the generation description fully realize the script's key events?") These are structured question instructions, rather than a harness system prompt. | Generated script + director prompt |
| **Editing** | Local Python heuristic; no model | No prompt. Scores storyboard shot count, mean shot duration and duration variation. Uses `planning.max_shots` and `planning.min_shot_seconds`. | Generated storyboard's shot list and durations |

Implementation sources in the sibling checkout: [video prompt, video_quality.py:14](../../rsi-1st-impl-design/rsi_harness/reward/video_quality.py#L14), [script-quality question, policies.py:64](../../rsi-1st-impl-design/rsi_harness/neohorse/policies.py#L64), [key-event consistency question, policies.py:75](../../rsi-1st-impl-design/rsi_harness/neohorse/policies.py#L75), and [editing heuristic, editing_quality.py:16](../../rsi-1st-impl-design/rsi_harness/reward/editing_quality.py#L16). The two story judgments are combined in [story_quality.py:35](../../rsi-1st-impl-design/rsi_harness/reward/story_quality.py#L35).

$$
S_{\mathrm{clip}} = R_{\mathrm{clip}} - P_{\mathrm{axis}}
$$

$$
P_{\mathrm{axis}} =
\begin{cases}
\displaystyle \sum_{a \in A} w_a \max\!\left(0,\ b_a - c_a - \tau\right),
& \text{if axis-regression protection is enabled},\\
0, & \text{otherwise}.
\end{cases}
$$

With protection enabled and zero tolerance, each axis loss has twice its weighted impact on S_clip relative to Base.

Protection is by default enabled, $\tau = 0$, and the weights are video 0.60, story 0.25 and editing 0.15:

$$
P_{\mathrm{axis}} =
0.60\max(0,b_{\mathrm{video}}-c_{\mathrm{video}})
+0.25\max(0,b_{\mathrm{story}}-c_{\mathrm{story}})
+0.15\max(0,b_{\mathrm{editing}}-c_{\mathrm{editing}}).
$$

Only regressions add a penalty; improvements on another axis do not cancel it. This definition follows `VideoGEPAAdapter.search_score` in `rsi_harness/optimization/gepa_adapter.py`; invalid evaluations receive search score zero.

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

**Project penalties.** S_project does not subtract the short-clip axis-regression penalty: C averages raw R_clip, not S_clip. The project scorer supports a separate within-clip loop-return penalty, but its weight is zero in the displayed run, so no penalty is subtracted from either project score here. Invalid projects receive score zero. See [ProjectScorePolicy](../../rsi-1st-impl-design/rsi_harness/optimization/project_adapter.py#L26).

**Visual handoff H.** For each adjacent clip pair, compare the earlier clip's last frame $L_i$ with the later clip's first frame $B_{i+1}$:

$$
h_i = 1-\frac{1}{255D}\sum_{p=1}^{D}
\left|L_i[p]-\widetilde{B}_{i+1}[p]\right|,
\qquad
H=\frac{1}{N-1}\sum_{i=1}^{N-1}h_i.
$$

$N$ is the number of clips; $D$ is the number of pixel-channel values in $L_i$ (height times width times three RGB channels). Pixel values range from 0 to 255. $\widetilde{B}_{i+1}$ is the next first frame, bilinearly resized to match $L_i$ if needed. A five-clip project averages four handoffs. The implementation rounds each $h_i$ to four decimals and averages available transition scores; if none are available, H is missing and the frozen missing-metric policy applies.

H ranges from 0 to 1: identical boundary frames score 1. It measures pixel similarity, not semantic story continuity, motion correctness or audio smoothness. First-frame conditioning can therefore make H high even when the later clip fails its requested action. See [frame_similarity](../../rsi-1st-impl-design/rsi_harness/longform/evaluator.py#L104) and [transition aggregation](../../rsi-1st-impl-design/rsi_harness/longform/evaluator.py#L218).

F is [state confirmation (state_confirmed_rate)](../../rsi-1st-impl-design/rsi_harness/longform/evaluator.py#L233), P is [payoff (payoff_rate)](../../rsi-1st-impl-design/rsi_harness/longform/evaluator.py#L235), and H is visual handoff: similarity between adjacent last/first frames. The [counting logic](../../rsi-1st-impl-design/rsi_harness/longform/evaluator.py#L194) computes F as confirmed facts / declared facts and P as passed / executed payoff checks, using a confirmation threshold of 0.5. Both project scores use the current project's metrics. The current longform implementation does not carry the short-clip axis-regression penalty into the project formula.

Train/Validation use `S_project` for project search. Independent Promotion uses newly evaluated held-out `R_project` and safeguards; Test reports its own `R_project`. R_project can also be recorded on search projects as a diagnostic without driving selection.

These project formulas assume all metrics are present and loop penalty is zero; missing metrics follow the frozen policy. A higher search score does not guarantee a higher reward/report score or better output in every case. Clip-level and project-level values have different definitions and cannot be compared directly. The demonstrated runs use these scores for harness search/evaluation, not direct model-weight updates.

For concrete examples, see the [single-clip evaluation](subdocs_examples/08-script-rewrite-prompt-lifecycle.md#10-evaluation-and-what-the-example-demonstrates) and the [five-clip reward aggregation and project evaluation](subdocs_examples/11-town-fair-longform-production.md#7-clip-rewards-and-project-evaluation). The latter maps recorded fields to clip and project metrics, and its two town-fair Train projects now carry their recorded per-segment and project values.

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
