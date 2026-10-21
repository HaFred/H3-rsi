# 03. Harness RSI: Generation, Evaluation, and Promotion

Source: PPTX page 3 of 17.

The main loop starts from a Base Harness with six text fields, generates real audio/video with H3, scores the output under a frozen evaluation policy, and uses GEPA to rewrite one component before validation and promotion.

```mermaid
flowchart LR
  A[Base Harness] --> B[H3 audio/video generation]
  B --> C[Frozen evaluation: R_clip and S_clip]
  C --> D[GEPA rewrites one text component]
  D --> E[Validation and independent promotion]
```

| Split | Role and feedback boundary |
|---|---|
| Train | Compare parent and child using S_clip; only Train trajectories reach reflection |
| Validation | Return scalar S_clip values, maintain the per-task Pareto frontier, select best by mean S_clip |
| Promotion | Independently reevaluate Base/best using R_clip and quality axes; results do not feed back into this search |
| Test | Report Base/best R_clip and axis scores; results do not influence selection |

For short clips, `R_clip` is the weighted reward measured on every evaluated case, including Train/Validation; `S_clip = R_clip - axis-regression penalty` is derived from that same search case. Selection uses S_clip, then independent Promotion reevaluates the winner on held-out cases using their own R_clip and quality safeguards. This is a decision sequence, not a dependency on future promotion scores. The searched components are script, storyboard, director, and memory. Critic/editor remain fixed diagnostic components. See [the split and scoring definitions](07-data-splits-and-scores.md).

---

[Contents](README.md) · [Previous](02-rsi-definition.md) · [Next](04-service-topology.md)
