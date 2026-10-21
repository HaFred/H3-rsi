# 06. GEPA: Search, Evaluation, and Promotion Loops

Source: PPTX page 6 of 17.

### Frontier construction and maintenance

In this project, GEPA's **instance-wise Pareto frontier** tracks the complete harnesses with the highest S_clip on each Validation case (val-001 and val-002), including ties. Its dimensions are Validation cases, not quality axes or harness components. Script, storyboard, director and memory rewrites all contribute to the same candidate pool and frontier.

**Initialization.** Evaluate Base on the full Validation set. With no other candidates evaluated, each case's leader set is {Base}, so the initial frontier contains only Base and the first parent must be Base.

**A new candidate joins the frontier if it beats or ties the best score on at least one Validation case.** It must first pass the Train parent/child comparison and receive full Validation scores. Update each case's leader set independently:

| New candidate's S_clip on a Validation case | Update to that case's leader set |
| --- | --- |
| Higher than the current best | Replace the previous leaders with the new candidate |
| Equal to the current best | Add the new candidate alongside the existing leaders |
| Lower than the current best | Leave the leader set unchanged |

The frontier candidates are the union of these per-case leader sets. A harness leaves the frontier when it no longer leads any case, but remains in the recorded candidate pool. Passing Train or improving the Validation mean alone does not guarantee frontier membership.

For example, if val-001 is led by A at 0.80 and val-002 by B at 0.78, a new C scoring 0.82 / 0.75 replaces A only. The frontier becomes {B, C}; A remains recorded. These numbers are illustrative.

**Parent selection versus final selection.** GEPA 0.1.4 prunes redundant tied leaders before parent sampling and weights surviving parents by the number of case-leader sets they occupy. The final winner is selected from the candidate pool by full-Validation mean S_clip; it need not lead an individual case. This per-case leadership structure is more selective than the general mathematical set of all nondominated score vectors. See the [versioned parent-selection implementation](https://github.com/gepa-ai/gepa/blob/v0.1.4/src/gepa/gepa_utils.py#L82).

### GEPA search and promotion steps

1. Sample a parent from the Pareto frontier according to its per-Validation-task S_clip leadership.
2. Select script, storyboard, director, or memory with `round_robin`.
3. Ask Qwen3.5-9B to rewrite the selected field using Train feedback only.
4. Construct the child Harness by replacing that one field and retaining the other fields from the parent.
5. Evaluate on the same two Train tasks. Continue only if the child's sum of S_clip is strictly greater than the parent's; otherwise discard it and select a parent again.
6. Evaluate the accepted child on the complete Validation set (`val-001`, `val-002`).
7. Add it to the candidate pool and update the per-task Pareto frontier using S_clip.
8. Repeat until the evaluation, H3-request, or wall-clock budget stops search.
9. Select the winner with the highest full-Validation mean S_clip.
10. Independently reevaluate Base/winner on six Promotion tasks using R_clip and quality axes. Require video-axis gain and regression safeguards.

Every candidate/task evaluation reruns Script -> Storyboard -> Director and generates video with frozen H3. The evaluator measures the current clip reward `R_clip`; search derives `S_clip = R_clip - axis-regression penalty` from that same case. Promotion later generates and scores independent held-out cases using their own `R_clip` and quality safeguards. R_clip is not confined to Promotion/Test, and search never uses their future scores. See [the scoring definitions](07-data-splits-and-scores.md).

If the gate accepts, promote the winner to champion and update `champion.txt`. If rejected, retain the existing champion; the search winner remains available without changing deployment. Champion inheritance as the seed of a later run is designed, but a second run was not demonstrated in this experiment.

---

[Contents](README.md) · [Previous](05-search-boundary.md) · [Next](07-data-splits-and-scores.md)
