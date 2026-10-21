# 08. Harness Improvements: Short-Clip Comparison

Source: PPTX page 8 of 17.

The same task, `train-001`, is evaluated with three Harness versions. `R_clip` is the underlying clip reward measured on this Train task; search derives S_clip from it as explained on [page 7](07-data-splits-and-scores.md).

> A person in a yellow raincoat picks up a red umbrella on a wet street. Keep the same person and umbrella throughout. Include footsteps and dripping water.

| Version | R_clip | Video axis |
|---|---:|---:|
| Base | 0.6348 | 0.477 |
| Rewrite script | 0.6628 | 0.541 |
| Then rewrite director | 0.6659 | 0.556 |

[Worked example: the complete script-rewrite prompt lifecycle](subdocs_examples/08-script-rewrite-prompt-lifecycle.md) shows this row's actual task, five agents' system/user messages and saved outputs, H3 request, and video.

The script rewrite adjusts shot timing. The director rewrite specifies the subject and appearance constraints more completely.

The Harness consists of reusable system prompts and memory. Scripts, storyboards, and director prompts are outputs for the current task. Reflection at `:8001` proposes candidate text; the framework validates and assembles it; production at `:8000` executes it. Evaluation and the gate determine acceptance and promotion.

Still images are provided below. The original animated preview links are retained separately from the local media.

### Source images (idx 48 just for vis)

![Page 8, image 1](assets/image2.png)

![Page 8, image 2](assets/image3.png)

![Page 8, image 3](assets/image1.png)

### Original animated previews

- [Base](https://raw.githubusercontent.com/HaFred/H3-rsi/main/evaluations/edb00ad03c2e77bc2b759d587f1a1f2c0706fcf90405d129cae6ee19daa4771b/preview-lg.webp)
- [Script rewrite](https://raw.githubusercontent.com/HaFred/H3-rsi/main/evaluations/0aa979018910a10da8a371e18edc03464f18690acf534c0948e629736dd44170/preview-lg.webp)
- [Then rewrite director](https://raw.githubusercontent.com/HaFred/H3-rsi/main/evaluations/93d21889efd4bd045da797839dc3add5758da0c7d097d3ca06c15a666ff43810/preview-lg.webp)

---

[Contents](README.md) · [Previous](07-data-splits-and-scores.md) · [Next](09-short-clip-videos.md)
