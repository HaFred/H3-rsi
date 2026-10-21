# 14. Long-Form Optimization: GEPA and Project Execution

Source: PPTX page 14 of 17.

### Outer loop: select a Harness using complete-project S_project

1. Select a parent from the Pareto frontier, retaining accepted candidate lineages.
2. Rotate through components and rewrite exactly one; inherit the other five text fields from the parent.
3. Evaluate Train projects by generating all five segments for each project.
4. Discard children without strict Train-score improvement. Evaluate passing children on the full Validation set and update the Pareto frontier.
5. At search completion, select the winner by mean Validation S_project, then submit it to the independent Promotion Gate.

### Inner loop: execute five segments sequentially

Project and confirmed facts → current-segment brief → sequential H3 generation (text for the first, first-frame conditioning thereafter) → visual evidence/payoff judgments → update context, withholding unconfirmed facts → concatenate and evaluate `timeline.mp4` and project metrics.

A negative fact judgment affects context and does not cause regeneration. RewardBuilder scores segment quality independently.

Only Train-project feedback reaches reflection: segment quality, unconfirmed facts, cut/handoff metrics, and component inputs/outputs. Fact diagnostics can inform reflection even though they are excluded from scalar S_project.

`S_project = 0.7 C + 0.3 H`, where `C = mean(R_clip)` across segments.

`R_project = 0.4 C + 0.2 F + 0.2 P + 0.2 H` additionally includes state confirmation and payoff. Both scores can be measured on the current project, including during search. S drives Train/Validation selection; independent Promotion evaluates R on held-out projects and Test reports it. These formulas assume all metrics are present and loop penalty is zero; the frozen policy defines missing-metric handling.

---

[Contents](README.md) · [Previous](13-town-fair-videos.md) · [Next](15-joint-optimization.md)
