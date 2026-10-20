# 10. How Harness Changes Propagate Downstream

Source: PPTX page 10 of 17.

Qwen3.5-9B at `:8001` proposes Harness changes. In this lineage only `prompts.script` and `prompts.director` change. neohorse-1-4B at `:8000` executes each candidate and regenerates production artifacts.

| Artifact length, characters | Base | Script rewrite | Subsequent director rewrite |
|---|---:|---:|---:|
| Script output | 770 | 851 | 1,048 |
| Storyboard input user message | 798 | 879 | 1,076 |
| Director input user message | 1,658 | 1,945 | 2,172 |
| Final H3 prompt | 2,872 | 4,588 | 4,009 |

These are generated artifact/input lengths, rather than Harness system-prompt lengths.

- **Task input:** unchanged task and target duration. The source's task “seed” means input text here, rather than a sampling RNG seed.
- **Script system prompt:** grows from 233 to 1,058 characters, specifying three shots, time ranges totaling five seconds, and repeated subject, clothing, and prop descriptions.
- **Storyboard system prompt:** stays at 266 characters. Its user input grows because it includes the new script. Shot durations move from 2/2/1 seconds toward approximately 1.67 seconds per shot.
- **Director system prompt:** grows from 311 to 3,512 characters. It repeats appearance constraints per shot, preserves timestamps, adds lighting/material/action detail, and includes positive/negative examples.
- **H3:** weights and generation parameters remain fixed. The regenerated director prompt expresses more detailed constraints. Quality, rather than text length, is the objective.

Observed video scores rise 0.477 → 0.541 → 0.556 and R rises 0.6348 → 0.6628 → 0.6659. Story decreases slightly while editing stays at 1.000; the result does not establish improvement in every quality dimension.

Each evaluation reruns the chain, and text sampling can change upstream outputs. A longer script output in the third version does not imply that the script system prompt was rewritten again. These observations do not isolate the causal contribution of every change.

---

[Contents](README.md) · [Previous](09-short-clip-videos.md) · [Next](11-long-form-production.md)
