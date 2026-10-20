# 06. GEPA: Search, Evaluation, and Promotion Loops

Source: PPTX page 6 of 17.

1. Sample a parent from the Pareto frontier according to its per-Validation-task S leadership.
2. Select script, storyboard, director, or memory with `round_robin`.
3. Ask Qwen3.5-9B to rewrite the selected field using Train feedback only.
4. Construct the child Harness by replacing that one field and retaining the other fields from the parent.
5. Evaluate on the same two Train tasks. Continue only if the child's sum of S is strictly greater than the parent's; otherwise discard it and select a parent again.
6. Evaluate the accepted child on the complete Validation set (`val-001`, `val-002`).
7. Add it to the candidate pool and update the per-task Pareto frontier using S.
8. Repeat until the evaluation, H3-request, or wall-clock budget stops search.
9. Select the winner with the highest full-Validation mean S.
10. Independently reevaluate Base/winner on six Promotion tasks using R and quality axes. Require video-axis gain and regression safeguards.

Every candidate/task evaluation reruns Script → Storyboard → Director, generates video with the frozen H3 model using the director output as its prompt, and computes R and S. For short clips, `S = R − axis-regression penalty`.

If the gate accepts, promote the winner to champion and update `champion.txt`. If rejected, retain the existing champion; the search winner remains available without changing deployment. Champion inheritance as the seed of a later run is designed, but a second run was not demonstrated in this experiment.

---

[Contents](README.md) · [Previous](05-search-boundary.md) · [Next](07-data-splits-and-scores.md)
