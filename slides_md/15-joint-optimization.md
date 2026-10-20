# 15. Harness and Model RSI: Parallel Optimization

Source: PPTX page 15 of 17.

This page describes the proposed joint-optimization design, rather than an outcome demonstrated by the frozen-model run.

Freeze a baseline snapshot `(H₀, θ₀, J)`, where H is the Harness, θ is the model checkpoint, and J is the evaluation/judging setup.

| Branch | Fixed input | Optimized output |
|---|---|---|
| GEPA | Model θ₀ | Harness H₁ |
| verl-omni | Harness H₀ | Model θ₁ |

Evaluate four combinations on Validation:

| Combination | Purpose |
|---|---|
| `(H₀, θ₀)` | Baseline |
| `(H₁, θ₀)` | Harness-only change |
| `(H₀, θ₁)` | Model-only change |
| `(H₁, θ₁)` | Combined change |

Select the combination using Validation, verify it with the independent Promotion Gate, and atomically promote the complete system version.

---

[Contents](README.md) · [Previous](14-long-form-optimization.md) · [Next](16-evidence-and-roadmap.md)
