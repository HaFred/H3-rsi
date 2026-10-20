# 11. Long-Form Production: State Verification and First-Frame Handoff

Source: PPTX page 11 of 17.

A project generates five segments sequentially. The visual judge controls which facts enter the next segment's context.

1. Load `ProjectSpec` and initial `StoryState S₀`.
2. Generate the first segment using t2va. Generate later segments using fl2va, conditioned only on the previous valid committed segment's last frame as their first frame.
3. Ask the visual judge at `:8090` whether new evidence is visible and whether previously confirmed facts remain visible as payoffs.
4. Confirm visible new facts; mark unconfirmed facts `withheld` and omit them from downstream context.
5. Continue through segment five, normalize with ffmpeg, and concatenate to `timeline.mp4`.
6. Independently compute each segment's quality R with RewardBuilder (video/story/editing), aggregate project quality, handoff and factual payoff, and archive video, state, and costs.

A negative evidence judgment only withholds a fact. It does not fail or regenerate the segment. Payoff judgments score previously confirmed facts without adding or deleting facts.

Invalid quality signals or parse/validation failures invalidate the segment and block dependent segments. Systemic service errors such as timeouts abort the run. These failure paths are separate from a negative fact judgment.

---

[Contents](README.md) · [Previous](10-prompt-propagation.md) · [Next](12-town-fair-lineage.md)
