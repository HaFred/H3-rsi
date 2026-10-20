# 05. The Harness Search Boundary

Source: PPTX page 5 of 17.

The Harness contains six configurable text components.

| Component | Meaning | Searched in this run? |
|---|---|---|
| `script` | Scriptwriter system prompt | Yes |
| `storyboard` | Storyboard system prompt | Yes |
| `director` | Director system prompt | Yes |
| `critic` | Critic system prompt | No; fixed diagnostics |
| `editor` | Editor system prompt | No; fixed diagnostics |
| `memory` | Shared experience text | Yes |

Model weights, workflow order, routing, generation parameters, reward weights, and judges remain fixed. Freezing the evaluation protocol constrains the search space and keeps comparisons interpretable.

Generated scripts, storyboard JSON, and final H3 prompts are task outputs derived from these fields, rather than additional directly searched Harness components.

---

[Contents](README.md) · [Previous](04-service-topology.md) · [Next](06-gepa-loops.md)
