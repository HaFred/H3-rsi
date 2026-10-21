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

### Why search and reporting use different scores

| Stage | Question | Score |
|---|---|---|
| Search | Which harness should we explore and select, while discouraging quality regressions? | `S_clip = R_clip − penalty` |
| Reporting | What quality did the selected harness actually achieve under the fixed evaluation rubric? | Raw `R_clip`, plus individual axes |
| Promotion | Is the selected harness good enough to replace the champion on independent cases? | Held-out `R_clip` and explicit safeguards |

For short clips, the penalty makes regressions more expensive during search; it guides selection without changing raw reward. Reporting `R_clip` preserves a fixed quality rubric across runs whose Base harnesses may differ. Report individual axes alongside it to expose tradeoffs. Both scores come from the current evaluation; search does not depend on future Promotion results.

Reporting raw reward does not undo selection by `S_clip` or authorize deployment. Independent Promotion checks held-out performance, axis and per-case regressions, and quality floors; a search winner can still fail to become champion. See [the split and scoring definitions](07-data-splits-and-scores.md).

The searched components are script, storyboard, director, and memory. Critic/editor remain fixed diagnostic components.

---

[Contents](README.md) · [Previous](02-rsi-definition.md) · [Next](04-service-topology.md)
