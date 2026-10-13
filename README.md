# RSI × MiniMax-H3 — end-to-end demo

This run shows the full loop, on real hardware:

1. the **base harness** drives the H3 server to generate the baseline video;
2. the **RSI framework** scores that video with a frozen policy and rewrites the harness prompts;
3. the **optimized harness** drives H3 again and produces a better video on the same task;
4. the framework **keeps iterating** — candidates that don't win the validation split are dropped — and the
   winner is gated on a held-out promotion split before it becomes the champion.

Run `101214_1strun`, 2026-10-12 14:20 → 16:54 UTC. **The optimized harness was accepted and promoted to
`gepa-base.1`.**

| Split (held-out) | Tasks | Base harness | Optimized harness | Δ |
| --- | --- | --- | --- | --- |
| Validation (search signal) | 2 | 0.6588 | **0.6749** | +0.0161 |
| Promotion gate | 6 | 0.6107 | **0.6528** | +0.0421 |
| Test (never used for any decision) | 4 | 0.5905 | **0.6393** | +0.0488 |

These are mean **total R** per split; section 2 breaks R down and names the model behind each part.
All numbers are only comparable **within this run** (same judge, same policy).

---

## 1. What was serving

| Role | Model / server | Endpoint | Where |
| --- | --- | --- | --- |
| Video generation | MiniMax-H3 fl2va, SGLang `450837e`, TP2, `--performance-mode memory` | `http://localhost:30010` | GPU 6,7 |
| Production text (runs the harness agents) | `neohorse-1-4B` (vLLM, thinking off) | `http://localhost:8000` | GPU 5 |
| Reflection text (GEPA mutator) | `Qwen/Qwen3.5-9B` | `http://127.0.0.1:8001` | GPU 5 |
| Decision (critic gates, story axis) | `NeoHorse-Jev-4B` | `http://localhost:8085` | GPU 5 |
| Frame judge (video axis, 60% of score) | `UnifiedReward-2.0-qwen3vl-2b` | `http://localhost:8090` | GPU 5 |

Every clip is H3 `t2va`: 1344×768, 16:9, 24 fps, 5 s, 10 inference steps, one candidate — about 50 s per clip.
Each evaluation = 1 H3 request, so the run billed **65 requests** (search 45/60, promotion 12/12, test 8/8).

RSI touches **none** of these services. It only edits six text fields in the harness config; the services and
the reward weights stay frozen.

---

## 2. How RSI improves the harness — watch these clips in order

**Scope.** This document covers the **text-only** half: H3 and the judges stay frozen while RSI searches the six
text components. The *harness-only* Context-IR path — a frozen compiler in front of H3 plus weight training
(actor GRPO + H3 DiffusionNFT/FlowGRPO), with **no harness search at all** — is
[verl-omni RFC #500](https://github.com/verl-project/verl-omni/issues/500). The RSI half is
[#748](https://github.com/verl-project/verl-omni/issues/748), which these clips support.

### What gets rewritten: six text components

The harness is a YAML config, but RSI can rewrite only six text fields inside it. Everything else — reward
weights, tool routing, workflow order, generation parameters — is inherited from the base and held fixed, so the
search cannot buy score by touching the rules instead of the prompts.

| Component | Its job (as written in the seed `base.yaml`) |
| --- | --- |
| `prompts.script` | screenwriter — expand the one-line idea into a short film script: setup → conflict → resolution |
| `prompts.storyboard` | storyboard artist — turn the script into a JSON shot list (`index`, `seconds`, `shot_size`, `camera`, `action`) |
| `prompts.director` | director — write the prompt MiniMax-H3 actually receives: `integrated_multimodal_description` (`[Shot N]` blocks with timestamps), `overall_soundscape`, `non_diegetic_music` |
| `prompts.critic` | film critic — list concrete defects under physics, subject consistency, story clarity, pacing, audio |
| `prompts.editor` | editor — pick the best candidate and say which shots to keep or shorten |
| `memory` | **not a prompt**: a shared list of heuristics appended to *every* agent's system prompt |

The five `prompts.*` entries are the system prompts of the five workflow agents (`script → storyboard →
director → critic → editor`); `memory` is appended to all of them. Those six fields are the entire search space
(see [Where the prompts live](#where-the-prompts-live) for what each one is at every step).

### What does the searching: GEPA

GEPA optimizes *textual* system components rather than model weights — instead of gradients it has an LLM reflect
on per-case feedback, and a Pareto front decide what survives. This run used `gepa` **0.1.4**, configured as:

| Setting | Value | What it means here |
| --- | --- | --- |
| the candidate | the six fields above | one candidate = one complete harness |
| the score | real end-to-end reward | a candidate is judged by driving H3 to generate an actual clip and running the frozen policy over it, so text that merely *looks* better cannot win |
| the proposer | reflection LM, `Qwen3.5-9B` on `:8001` | shown the current component text plus that minibatch's per-case feedback, and asked to rewrite **one** component |
| `candidate_selection_strategy` | `pareto` | selection is from the Pareto front, which keeps every candidate that is best on **at least one** case — so a component that helps one task but not another is not discarded |
| `module_selector` | `round_robin` | which of the six components to rewrite cycles, rather than always the same one |
| `reflection_minibatch_size` | `2` | a proposal is first tried on 2 tasks |
| `max_metric_calls` | `60` | search budget, in evaluator calls |
| `use_merge` | `false` | GEPA's option to combine components from different candidates is off |

Each iteration therefore runs the same four steps: **pick** a program off the Pareto front → **rewrite** one
component with the reflection LM → **score** the candidate on a 2-task minibatch → if it wins there, **validate**
it on real H3 clips and update the front. This run took 13 iterations; the per-iteration ledger is in §4.

The mechanism in one picture. RSI is a loop around H3; it never changes H3, only the text H3 is called with:

```text
   base harness text ──H3──▶ video V0 ──frozen policy──▶ S0 = 0.6588  (mean total R, validation split)
        ▲                                                       │
        │                                            per-case feedback:
        │                                            judge scores, shot pacing,
        │                                            critic gates, Jev story scores
        │                                                       ▼
   new harness text ◀──── GEPA + reflection LM (Qwen3.5-9B) rewrites ONE component
        │
        └──H3──▶ video V1 ──frozen policy──▶ S1
                       S1 > S0 ?  yes → becomes the new best, iterate from it
                                  no  → discard, propose a different component
                                                  │
                          search ends ────────────┘
                                                  ▼
                       6 unseen tasks → promotion gate → champion `gepa-base.1`
```

GEPA compares candidates on `S − Σ w_axis·max(0, base_axis − cand_axis)` so that video gains cannot be bought
with story losses; S itself is the plain mean of total R. See [S is total R](#s-is-total-r--but-gepa-searches-a-penalized-form-of-it).

The clips below are the **same task** (`train-001`: *"A person in a yellow raincoat picks up a red umbrella on a
wet street. Keep the same person and umbrella throughout. Include footsteps and dripping water."*), rendered
three times as RSI edits the harness. Work down the clip table — each row is the same task rendered again, so
consecutive rows show what the next edit changed.

### Where each number comes from

Every score in this document is produced by the evaluator, not by hand. Three of the four axes come from a
served model; `editing` is computed in code from the storyboard JSON and uses no model at all.

| Column | Produced by | Endpoint | Formula |
| --- | --- | --- | --- |
| **Total R** | `rsi_harness/reward/builder.py` | none (aggregation) | `0.60·video + 0.25·story + 0.15·editing` (fixed weights, from the frozen evaluation policy) |
| **video** | `UnifiedReward-2.0-qwen3vl-2b` — the frame judge | `http://localhost:8090` | the clip is 124 frames (5.17 s @ 24 fps); every 24th frame is sampled → 6 frames; per frame the judge rates **alignment, coherence, style** each 1–5 → `((a+c+s)/3 − 1)/4` → mean over the 6 frames |
| **story** | `NeoHorse-Jev-4B` — the decision model | `http://localhost:8085` | `0.6 · completeness(script) + 0.4 · consistency(script, director_prompt)` |
| **editing** | `rsi_harness/reward/editing_quality.py` | **no model** | `0.45` base, `+0.20` if 2 ≤ shots ≤ max_shots+2, `+0.20` if mean shot length is in range, `+0.15` if `length_cv ≤ 0.35` (−0.15 if ≥ 0.8); clamped to 0–1 |
| **last-frame alignment** | same judge, scalar diagnostic | `http://localhost:8090` | the raw 1–5 **Alignment** score of the **final sampled frame only** (the evaluator stores just the last judge response) — read it as a hint, not an aggregate |
| **S** (S0/S1 in the diagram) | `rsi_harness/optimization/gepa_adapter.py` | none (aggregation) | mean of total R over the validation split. S0 = (0.64337 + 0.674278)/2 = **0.6588** for the base; S1 = (0.657774 + 0.692103)/2 = **0.6749** for the winner |
| **search score** (what GEPA optimizes) | `VideoGEPAAdapter.search_score` | none (aggregation) | total R **minus an axis-regression penalty**: `R − Σ w_axis · max(0, base_axis − cand_axis)` per case, using `axis_weights` = the fixed policy weights (video .60, story .25, editing .15) |

The judge always compares the frames against the **original one-line task prompt** (`task.prompt`), never against
the director's rewritten prompt. So the video axis measures "does this look like the task", not "does this match
the harness's own prose".

### S is total R — but GEPA searches a penalized form of it

This distinction is easy to miss, so it is worth stating plainly. S0/S1 in the diagram **are** means of total R,
and for this run they are also exactly what GEPA recorded, because the base program trivially has zero penalty
and the winning candidate happened to regress no axis. But GEPA's search is not driven by raw total R.

`run_config.resolved.yaml` for this run set `guard_axis_regression: true` with `axis_regression_tolerance: 0.0`,
so per case GEPA sees `R − Σ w_axis·max(0, base_axis − cand_axis)`. It cannot buy video score by sacrificing
story score. The penalty reproduces GEPA's own logged per-case numbers exactly:

| Candidate | Case | total R | penalty | what GEPA optimized |
| --- | --- | --- | --- | --- |
| `script` v1 (`d58fde46`) | val-001 | 0.632439 | −0.012797 (story dipped) | 0.619642 |
| `script` v1 (`d58fde46`) | val-002 | 0.693740 | 0 | 0.693740 |
| `storyboard` (`db013ff2`) | val-001 | 0.542401 | −0.100968 | 0.441433 |
| `storyboard` (`db013ff2`) | val-002 | 0.648248 | −0.034896 | 0.613352 |
| **winner** (`4ddd73aa`) | both | 0.657774 / 0.692103 | 0 | 0.657774 / 0.692103 |

So: total R is the reporting metric and the promotion metric; the search metric is total R with axis regressions
charged against it. Reported search scores in `search_report.json` (`base_validation_mean`,
`best_validation_mean`) are raw total R means taken from the evaluator's outcomes, which is why they read as the
S0/S1 values in the diagram.

Because each metric has exactly one producer, naming the producer **inside the column header** says it for every
row — a separate per-row "source" column would repeat identical text down the table.

| Step | What RSI changed in the harness | What the harness then asked H3 for | Total R | video · *UnifiedReward* `:8090` | story · *Jev* `:8085` | editing · *local, no model* | last-frame alignment · 1–5 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **0. Base** | — (seed harness) | 3 shots of **2.0 / 2.0 / 1.0 s** | 0.6348 | 0.477 | 0.793 | 1.000 | 2.87 |
| **1. after `script`** | timed shot list instead of a free script | 3 shots of **1.67 / 1.67 / 1.67 s** | 0.6628 | 0.541 (**+0.064**) | 0.753 | 1.000 | 2.87 |
| **2. after `director`** | full subject/appearance spec per shot | same 1.7 / 1.7 / 1.6 s, richer shot text | 0.6659 | 0.556 (**+0.015**) | 0.730 | 1.000 | 3.14 |

**The three renders.** Same rows as above, in a table of their own so the preview gets the full column width — click any clip for the full-rate video.

| Step | Clip — `train-001` |
| --- | --- |
| **0. Base** | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/edb00ad03c2e77bc2b759d587f1a1f2c0706fcf90405d129cae6ee19daa4771b/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/edb00ad03c2e77bc2b759d587f1a1f2c0706fcf90405d129cae6ee19daa4771b/preview-lg.webp" width="720" alt="base train-001"></a> |
| **1. after `script`** | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/0aa979018910a10da8a371e18edc03464f18690acf534c0948e629736dd44170/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/0aa979018910a10da8a371e18edc03464f18690acf534c0948e629736dd44170/preview-lg.webp" width="720" alt="+script train-001"></a> |
| **2. after `director`** | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/93d21889efd4bd045da797839dc3add5758da0c7d097d3ca06c15a666ff43810/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/93d21889efd4bd045da797839dc3add5758da0c7d097d3ca06c15a666ff43810/preview-lg.webp" width="720" alt="+director train-001"></a> |

> **About the clips.** Every clip cell in this document plays **inline**: an animated WebP preview that loops,
> linked to the full-rate 124-frame `video.mp4` (click to open GitHub's player). GitHub's markdown sanitizer strips
> `<video>`/`<iframe>` tags, so an animated image is the only way to get motion inside a README. Each narrative clip
> now sits in its own two-column table, so the text columns can no longer squeeze it: those use `preview-lg.webp` at
> **720 px** (previews are still generated at 480 px for the denser tables — `170 px` for the base/optimized pairs,
> `190 px` for the iteration ledger).
>
> **Where the prompts are.** The three renders above come from one task but three harness versions, so three
> different prompts produced them — see [Where the prompts live](#where-the-prompts-live).

**Harness Delta for the Steps**

The tables above show the *effect*; these are the harness documents that produced it. The harness is a YAML
config — the same structure as the codebase seed `configs/harness_gepa_base.yaml` (version `gepa-base`) — and
across the three steps **only two fields change**: `prompts.script` and `prompts.director`. Every other field
(`prompts.storyboard`, `prompts.critic`, `prompts.editor`, `memory`, `workflow`, `critic`, `reward_weights`,
`target_defaults`) is byte-identical to the seed at every step, which is why the `storyboard` and `memory`
candidates that won a minibatch never survived the full validation run (§4).

| Step | Full harness document | `prompts.script` | `prompts.director` |
| --- | --- | --- | --- |
| **0. base** | [`base.yaml`](https://github.com/HaFred/H3-rsi/blob/main/base.yaml) — `version: gepa-base`, equals the seed | 233 ch | 311 ch |
| **1. +`script`** | [`candidates/d58fde46…/harness.yaml`](https://github.com/HaFred/H3-rsi/blob/main/candidates/d58fde464b7f6e72b223770db9e3f50d6d9aa4babb138b75d4c77d27fadda30f/harness.yaml) | **1,058 ch** (rewritten) | 311 ch (unchanged) |
| **2. +`director`** | [`candidates/4ddd73aa…/harness.yaml`](https://github.com/HaFred/H3-rsi/blob/main/candidates/4ddd73aa8fa203c6aaef003092483f3f4d4c90210a62dfc352b1cd7f862b5f5d/harness.yaml) | 1,058 ch | **3,512 ch** (rewritten) |

Step 2 is the search winner, and the gate accepted it (§3): [`promotion/decision.json`](https://github.com/HaFred/H3-rsi/blob/main/promotion/decision.json)
records `promoted_version: gepa-base.1`. Two files carry the outcome — byte-identical in `prompts` and `memory`,
differing only in metadata:

- [`best.yaml`](https://github.com/HaFred/H3-rsi/blob/main/best.yaml) — the run's **search winner** snapshot: `version: gepa-base-gepa-4ddd73aa`,
  `parent_version: gepa-base`, `notes: … not promoted.`
- [`champion/gepa-base.1.yaml`](https://github.com/HaFred/H3-rsi/blob/main/champion/gepa-base.1.yaml) — the **promoted champion** written to the persistent
  store: `version: gepa-base.1`, `notes: Promoted from GEPA run 101214_1strun…`. The store's
  [`champion.txt`](https://github.com/HaFred/H3-rsi/blob/main/champion/champion.txt) points at it, and that pointer is what the next run starts from.

Each step also ships a machine-readable component map at `candidates/<hash>/candidate.json`.

<details>
<summary><b>0. base</b> — <code>prompts.script</code> (233 ch) and <code>prompts.director</code> (311 ch)</summary>

`prompts.script`
```text
You are a screenwriter. Given a one-line idea, expand it into a short film script
(<= 120 words) with a clear setup, a visible conflict and a resolution.
Keep a single protagonist and a single location unless the idea requires more.
```

`prompts.director`
```text
You are a film director writing a prompt for MiniMax-H3. Produce exactly three parts:
`integrated_multimodal_description` (with [Shot N] blocks and timestamps),
`overall_soundscape`, and `non_diegetic_music`. Be concrete about lighting, materials,
and motion. Repeat the protagonist's appearance in every shot.
```

</details>

<details>
<summary><b>1. +<code>script</code></b> — <code>prompts.script</code> (1,058 ch); <code>prompts.director</code> unchanged from base</summary>

`prompts.script`
```text
You are a Text-to-Video Script Specialist. Given a one-line idea and a target duration, expand it into a short video script optimized for visual generation models.

1. **Structure**: Generate exactly 3 shots. Use the format `**SHOT N (Start-Time-End-Time)**`. The sum of shot durations must equal the target duration precisely (e.g., 5.0s).
2. **Consistency**: Maintain strict visual consistency for the protagonist, clothing, and key props across all shots. Describe the subject identically in each shot to ensure alignment.
3. **Visuals**: Focus on concrete visual descriptors (lighting, color, texture, motion) rather than abstract concepts. Include environmental details (weather, time of day) that aid visual generation.
4. **Narrative**: Ensure the script arc is visible: Shot 1 (Setup), Shot 2 (Conflict/Action), Shot 3 (Resolution).
5. **Physics**: Ensure all physical actions are plausible and grounded to avoid generation errors.
6. **Constraints**: Total word count must be <= 120 words. Keep the location single unless the idea requires multiple.
```

</details>

<details>
<summary><b>2. +<code>director</code></b> — <code>prompts.director</code> (3,512 ch); <code>prompts.script</code> as in step 1</summary>

`prompts.director`
```text
You are an expert Film Director and AI Video Prompt Engineer specializing in the MiniMax-H3 video generation model. Your goal is to convert a provided script and storyboard into a highly detailed, production-ready prompt that ensures visual coherence, audio fidelity, and strict adherence to the video model's requirements.

## Task Objective
Generate exactly three distinct parts based on the input script and storyboard:
1. `integrated_multimodal_description`
2. `overall_soundscape`
3. `non_diegetic_music`

## Output Format Constraints
- Do not include any introductory or concluding text outside the three required sections.
- Use Markdown headers or clear labels for the three sections.
- **Timestamps:** You must use the exact start and end timestamps provided in the input script/storyboard. Do not alter the total duration or shot lengths.
- **Shot Structure:** In `integrated_multimodal_description`, wrap each shot description in a block starting with `[Shot N]` followed by the timestamp range.

## Critical Guidelines for High-Quality Output

### 1. Visual Consistency (Protagonist Continuity)
- **Mandatory Requirement:** You must identify the protagonist's full physical appearance (clothing, accessories, hair, physical traits) from the first shot and explicitly **repeat this description in every single shot** of the `integrated_multimodal_description`.
- **Reason:** This is critical for MiniMax-H3 to maintain subject consistency across the generated video sequence. Vague references like "the person" are insufficient; use specific descriptors (e.g., "flour-dusted sleeves and white apron", "bright yellow raincoat with black zippers").

### 2. Concrete Visual Details
- Describe **Lighting:** Specify color temperature (e.g., "warm tungsten," "cool blue moonlight"), source (e.g., "streetlamp glow," "oven interior"), and intensity.
- Describe **Materials:** Use textural specifics (e.g., "brushed steel," "dark polished oak," "slick cobblestone," "matte black zippers").
- Describe **Motion:** Define the physics of movement (e.g., "steam rises visibly in a thick plume," "rhythmic splashing footsteps," "camera slowly pulls back").

### 3. Audio Design
- **`overall_soundscape`:** Focus on diegetic audio. Include ambient room tone, environmental effects (rain, wind, fire), and Foley sounds (footsteps, mechanical creaks, opening doors) that match the visual action.
- **`non_diegetic_music`:** Focus on the emotional and narrative layer. Describe the genre, instrumentation, tempo, and mood (e.g., "low, resonant hum," "swelling orchestral swell," "ambient drone").

### 4. Alignment & Coherence
- Ensure the narrative flow in the visual description matches the `script` and `storyboard` actions exactly. Do not add new plot points or change the sequence of events.
- Ensure the total duration calculated from your timestamps matches the sum of the durations in the input script/storyboard.

## Input Data Usage
- Analyze the `script` for action and timing.
- Analyze the `storyboard` for camera angles (e.g., close, medium, wide) and shot types.
- Integrate these details into the visual descriptions to guide the camera work and framing.

## Examples of Good Practice
- **Bad:** "A baker opens the oven." (Too vague on appearance/lighting)
- **Good:** "[Shot 1] 0.0s - 1.6s [Close-up] A rustic brushed steel oven door... A baker, wearing a pristine white apron stained at the hem and sleeves dusted with fine white flour, turns the handle." (Specific appearance + Material + Lighting)
```

</details>



**Where these texts came from**

Step 0 is the seed, hard-coded in the codebase (`configs/harness_gepa_base.yaml`, with the same defaults in
`rsi_harness/config.py`). Steps 1 and 2 are **not** in the codebase — the strings `Text-to-Video Script
Specialist` and `expert Film Director and AI Video Prompt Engineer` appear nowhere under `configs/` or
`rsi_harness/`, only in this run's log and outputs. They were written at runtime by the reflection LM, and they
are the only components GEPA changed.

Every LM call is recorded in [`events.jsonl`](https://github.com/HaFred/H3-rsi/blob/main/events.jsonl) as a `reflection_call`; the two that produced these
steps line up with the stored text to the character once the code fences are stripped:

| Step | `reflection_call` | `response_chars` | Stored prompt | Gap |
| --- | --- | --- | --- | --- |
| 1. +`script` | call 1 · 14:26:42 | 1,068 | 1,058 | 10 — the ```` ``` ```` fence |
| 2. +`director` | call 10 · 14:52:36 | 3,530 | 3,512 | 18 — the ```` ```markdown ```` fence |

Thirteen calls were made in all; GEPA kept five as candidates and judged them purely on measured reward —
iterations 1, 4 and 7 passed the minibatch and were then rejected on full validation. So these deltas are
**proposed by the LM and selected by the evaluator**, not transcribed from anywhere. That is what makes the loop
in §1 a real search rather than a staged one.

Two scope limits are worth stating plainly, since they bound how much the word "self-improving" should carry.
The candidate codec (`rsi_harness/optimization/codec.py`) lets the LM change **only** the six text fields
(`prompts.{script,storyboard,director,critic,editor}` and `memory`); reward weights, routing, workflow and
generation parameters are inherited from the base and cannot be searched at all. And `memory` — the field that
would accumulate state *within* a run — won two minibatches but never a full validation, so the champion's
`memory` is empty. What actually recurs here is the harness version: the promoted champion seeds the next run.

**What to look for, in order:**

- **Base → +script:** the base harness has no duration constraint, so the script agent split the 5 s as 2/2/1 and
  H3's third shot runs for only one second — the action is clipped off. After the `script` edit the three shots
  are 1.67 s each and the beat lands. The **video axis jumps +0.064**; the judge's per-frame *coherence* term
  rises while its last-frame alignment is unchanged (2.87) — timing, not content, was the constraint.
- **+script → +director:** pacing is already fixed, so this edit buys something different — the director prompt
  now forces the protagonist's concrete appearance (yellow raincoat, white hood, black leggings, black sneakers)
  into **every** `[Shot N]` block. The video axis gains another +0.015 and the last-frame alignment moves
  2.87 → 3.14. Story dips slightly (0.753 → 0.730) — the elaborated prompt spends words on appearance that the
  Jev completeness scorer would rather see spent on action.
- Two edits, two different sub-signals, two different parts of the score. That is the whole point: RSI did not
  get a better video model or a kinder judge, it found the two text fields that were limiting H3 here.
- `editing` sits at 1.000 in all three renders, so it contributes a constant `0.15` and explains none of the
  movement. On this pair, only `video` (and a small `story` drift) moved the total.

The same progression repeats on `train-003` (`"A child in a green jacket catches a fallen paper kite…"`):

| Step | Shot lengths | Total R | video · *UnifiedReward* | story · *Jev* | editing · *local* | last-frame alignment |
| --- | --- | --- | --- | --- | --- | --- |
| Base | 2.0 / 1.0 / 2.0 s | 0.6348 | 0.533 | 0.661 | 1.000 | 2.87 |
| after `script` | 1.6 / 1.6 / 1.8 s | 0.6664 | 0.547 | 0.753 | 1.000 | 3.00 |
| after `director` | 1.6 / 1.6 / 1.8 s | 0.6773 | 0.571 | 0.739 | 1.000 | 3.12 |

**The three renders.** Same rows as above, unsqueezed.

| Step | Clip — `train-003` |
| --- | --- |
| Base | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/1d567d7950bb08c98d03069d0ce5ab05f02e455a2d18cbd816242214118a1b94/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/1d567d7950bb08c98d03069d0ce5ab05f02e455a2d18cbd816242214118a1b94/preview-lg.webp" width="720" alt="base train-003"></a> |
| after `script` | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/228a7fe756a8c04f2b7dc6d27c9a485d8eec263396087c8d909683d125afd3c9/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/228a7fe756a8c04f2b7dc6d27c9a485d8eec263396087c8d909683d125afd3c9/preview-lg.webp" width="720" alt="+script train-003"></a> |
| after `director` | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/6fdaee36ff9e3547feee88d2bdd89dfa8011085010805314e1062370e3c58a08/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/6fdaee36ff9e3547feee88d2bdd89dfa8011085010805314e1062370e3c58a08/preview-lg.webp" width="720" alt="+director train-003"></a> |

---

## 3. The two edits behind the win

**Edit 1 — `script`** (233 → 1,058 chars). The base prompt never mentioned timing; the rewrite makes timing the
first requirement and pins the output format.

| | prompt |
| --- | --- |
| before | *"Given a one-line idea, expand it into a short film script (<=120 words) with a clear setup, a visible conflict and a resolution. Keep a single protagonist and a single location…"* |
| after | *"Generate exactly **3** shots. Use the format `**SHOT N (Start-End)**`. **The sum of shot durations must equal the target duration precisely.** Maintain strict visual consistency for the protagonist, clothing and key props across all shots. Focus on concrete visual descriptors (lighting, colour, texture, motion)… Shot 1 Setup, Shot 2 Conflict, Shot 3 Resolution."* |

Observed effect: shot split 2/2/1 → 1.67/1.67/1.67 on train-001 and 2/1/2 → 1.6/1.6/1.8 on train-003; the
shot-length coefficient of variation drops from **0.283 to ≤0.075**.

**Edit 2 — `director`** (311 → 3,512 chars). The base prompt already said "repeat the protagonist's appearance",
so the rewrite **defines what that means** and adds worked examples, keeping the same three-part output contract.

| | prompt |
| --- | --- |
| before | *"Produce exactly three parts: `integrated_multimodal_description` (with [Shot N] blocks and timestamps), `overall_soundscape`, `non_diegetic_music`. Be concrete about lighting, materials, and motion. Repeat the protagonist's appearance in every shot."* |
| after | *"…**identify the protagonist's full physical appearance (clothing, accessories, hair, physical traits) from the first shot and explicitly repeat this description in every single shot**. Vague references like 'the person' are insufficient; use specific descriptors (e.g. 'bright yellow raincoat with black zippers'). Specify lighting colour temperature and source, materials, and the physics of motion. Example — Bad: 'A baker opens the oven.' Good: '[Shot 1] 0.0s-1.6s [Close-up] … A baker, wearing a pristine white apron stained at the hem…' "*

Observed effect: judge alignment 2.87 → 3.14 (train-001), 3.00 → 3.12 (train-003), 2.51 → 3.00 (test-001).

`storyboard`, `critic`, `editor` and `memory` were never selected, so the champion differs from the base in these
two fields only.

### The result on unseen data

Every one of the six promotion tasks improved, and the gate required the **video** axis to gain:

| Task | Base clip | Optimized clip | total R | video · *UnifiedReward* `:8090` | story · *Jev* `:8085` |
| --- | --- | --- | --- | --- | --- |
| promotion-001 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/ee06ed72226772933316f2622235b92f33d78d599504f40934105e3539f217a3/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/ee06ed72226772933316f2622235b92f33d78d599504f40934105e3539f217a3/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/9d747a8a7c3bd2969a32c381f1028569f94103ba534b3a798352f7f4fae9db7f/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/9d747a8a7c3bd2969a32c381f1028569f94103ba534b3a798352f7f4fae9db7f/preview.webp" width="170" alt="optimized"></a> | 0.637 → 0.653 | 0.525 → 0.548 (+0.023) | 0.688 → 0.698 |
| promotion-002 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/8d0f9415cf512c7700575fb55ac5bc352a45b155218090862277f7479e7657d7/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/8d0f9415cf512c7700575fb55ac5bc352a45b155218090862277f7479e7657d7/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/aeea3f5b03090fe0e176ffcc791fd692a04fa97d8e5ff382365c8e48961349ea/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/aeea3f5b03090fe0e176ffcc791fd692a04fa97d8e5ff382365c8e48961349ea/preview.webp" width="170" alt="optimized"></a> | 0.569 → 0.653 | 0.408 → 0.541 (**+0.133**) | 0.695 → 0.712 |
| promotion-003 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/a72559df839466f7c3006f8e9246e6b218f5c89654b732aa63841d88f264ae89/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/a72559df839466f7c3006f8e9246e6b218f5c89654b732aa63841d88f264ae89/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/2679672eb6afee820a028827abfd9bcdb578a008221f0a8726d079ad921c220d/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/2679672eb6afee820a028827abfd9bcdb578a008221f0a8726d079ad921c220d/preview.webp" width="170" alt="optimized"></a> | 0.593 → 0.652 | 0.461 → 0.523 (+0.062) | 0.666 → 0.752 |
| promotion-004 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/9fc6e25b1033dedd919899ad08994fc48e2927451757f7fd3b957049c22f9b36/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/9fc6e25b1033dedd919899ad08994fc48e2927451757f7fd3b957049c22f9b36/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/6e9fcf99e7fb99f6f97ededd244e5d91fefc526f1fe063093b698b51a658c857/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/6e9fcf99e7fb99f6f97ededd244e5d91fefc526f1fe063093b698b51a658c857/preview.webp" width="170" alt="optimized"></a> | 0.604 → 0.659 | 0.450 → 0.528 (+0.078) | 0.737 → 0.769 |
| promotion-005 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/1a698636c4ca67eb40cf931758bbf6c7a7a593b94ed669ce9b152aa3e562f2d1/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/1a698636c4ca67eb40cf931758bbf6c7a7a593b94ed669ce9b152aa3e562f2d1/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/0ddd2aa3ba40227594e21c5e7e9f6fbcdb969a758a01412e679f2ca060b95098/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/0ddd2aa3ba40227594e21c5e7e9f6fbcdb969a758a01412e679f2ca060b95098/preview.webp" width="170" alt="optimized"></a> | 0.639 → 0.635 | 0.460 → 0.469 (+0.009) | 0.852 → 0.814 |
| promotion-006 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/26f615b93539a55d59d0b17d118a819f6f70778afd4e5b213ad1f1a7deb8819b/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/26f615b93539a55d59d0b17d118a819f6f70778afd4e5b213ad1f1a7deb8819b/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/d83296894e58c66d069a7285ffc49b69f816bcd8c68bc9ecab2928abe51de19e/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/d83296894e58c66d069a7285ffc49b69f816bcd8c68bc9ecab2928abe51de19e/preview.webp" width="170" alt="optimized"></a> | 0.622 → 0.665 | 0.521 → 0.551 (+0.029) | 0.637 → 0.737 |
| **mean** | | | **0.611 → 0.653** | **0.471 → 0.527 (+0.056)** | 0.712 → 0.747 |

The video axis improved on **6 of 6** promotion tasks, which is what the gate tests. The one total that slipped,
promotion-005 (0.639 → 0.635), slipped on `story` (0.852 → 0.814), not on video — that task's base script was
already strong and the rewrite traded a little story completeness for a small visual gain.

Test split, never used for any decision:

| Task | Base clip | Optimized clip | total R |
| --- | --- | --- | --- |
| test-001 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/b5b924300fa39ee10c7b1359d7c17a4428a0e201681905b246661c872a6ebd95/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/b5b924300fa39ee10c7b1359d7c17a4428a0e201681905b246661c872a6ebd95/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/998b4be4e3bc7ed0294ed65a9c7fe78dc79c8dbcd480970f849084bbd25565f3/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/998b4be4e3bc7ed0294ed65a9c7fe78dc79c8dbcd480970f849084bbd25565f3/preview.webp" width="170" alt="optimized"></a> | 0.646 → 0.688 |
| test-002 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/46da8e301312ae856e5e163fa6a14628358f86a5f4dea66074dd395b73090357/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/46da8e301312ae856e5e163fa6a14628358f86a5f4dea66074dd395b73090357/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/4dc7b1061308e4d5524ce111e7a3a20a147b83d976334af78d8414cea3604235/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/4dc7b1061308e4d5524ce111e7a3a20a147b83d976334af78d8414cea3604235/preview.webp" width="170" alt="optimized"></a> | 0.509 → 0.607 |
| test-003 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/c623acf1696f096723dd27510456ce4ab2752f0f6b6f60420c76eb236a80df82/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/c623acf1696f096723dd27510456ce4ab2752f0f6b6f60420c76eb236a80df82/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/04f66ccd0b8f5c15bcf2aacef9db58776ecc47e5dd96b0fe273d5e0652588d69/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/04f66ccd0b8f5c15bcf2aacef9db58776ecc47e5dd96b0fe273d5e0652588d69/preview.webp" width="170" alt="optimized"></a> | 0.588 → 0.616 |
| test-004 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/0c9be593fa7a48bcabc65c1eedd31898ec8377c33a582e59016fb8a48a3b6b0a/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/0c9be593fa7a48bcabc65c1eedd31898ec8377c33a582e59016fb8a48a3b6b0a/preview.webp" width="170" alt="base"></a> | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/947bfd02d7e2c7eb16c6ba361da2c6a9b7c7bf9ef5663f829f09deee7695782b/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/947bfd02d7e2c7eb16c6ba361da2c6a9b7c7bf9ef5663f829f09deee7695782b/preview.webp" width="170" alt="optimized"></a> | 0.619 → 0.646 |
| **mean** | | | **0.590 → 0.639** |

### Where the prompts live

A clip is produced by three different kinds of prompt, and it helps to keep them apart:

| Prompt | What it is | Where to find it |
| --- | --- | --- |
| **Task prompt** — the seed | the one-line idea: the user prompt handed to the script agent, and the text the visual judge scores the clip against | the codebase's `benchmarks/tasks_gepa_v1.jsonl`. This repo carries the same text verbatim inside every `evaluations/<key>/outcome.json`, as the first line of `feedback` (`Task requirement: …`) |
| **System prompts** — the harness | the six text components RSI searches, `prompts.{script,storyboard,director,critic,editor}` and `memory` | [`base.yaml`](https://github.com/HaFred/H3-rsi/blob/main/base.yaml) (seed), [`best.yaml`](https://github.com/HaFred/H3-rsi/blob/main/best.yaml), [`champion/gepa-base.1.yaml`](https://github.com/HaFred/H3-rsi/blob/main/champion/gepa-base.1.yaml), and each `candidates/<hash>/harness.yaml` |
| **Derived prompts** — per clip | what the storyboard and director agents were actually asked, and the final prompt sent to H3 | each `evaluations/<key>/trajectory.json` — `script`, `storyboard`, `director_prompt` |

**Only the seed is shared between steps; everything downstream of it is rewritten.** For `train-001` the script
agent's user prompt is built from the task and the target alone — `Idea: <task prompt>` plus
`Target duration: 5s across 3 shots.` — and never reads the harness. Every later prompt is assembled from the
previous agent's output, so the sizes move at every step:

| Prompt | base | +`script` | +`director` | identical? |
| --- | --- | --- | --- | --- |
| script agent user prompt (the seed) | same | same | same | **yes** |
| script agent output | 770 ch | 851 ch | 1,048 ch | no |
| storyboard agent user prompt | 798 ch | 879 ch | 1,076 ch | no |
| director agent user prompt | 1,658 ch | 1,945 ch | 2,172 ch | no |
| **H3 prompt** (`director_prompt`) | 2,872 ch | 4,588 ch | 4,009 ch | no |

So "same prompt or rewritten?" is **both**: the seed is identical, and one edit to `prompts.script` propagates the
whole way down — the script text changes, which changes the storyboard agent's user prompt, which changes the
storyboard, which changes the director's user prompt, which changes what H3 is asked. The agent user prompts are
not stored; the rows above are reconstructed from the stored `script` and `storyboard` using the exact templates
in `rsi_harness/agents/*.py`.

---

## 4. The loop in detail (13 iterations)

Step 4 of your flow — "keep iterating" — is this ledger. GEPA picks a program, the reflection model rewrites
**one** component, the candidate is tried on a 2-task minibatch, and only if it wins there does it get a full
validation run (2 real H3 clips).

| Iter | Selected program | Component proposed | Minibatch | Full val | Val score | Clip from that iteration |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | — | — (base program) | — | yes | 0.6588 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/2c86b2f9b7b60ba40a8e9a49bf815b70cdb0e4b4b27c365d2a386ef5107cd854/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/2c86b2f9b7b60ba40a8e9a49bf815b70cdb0e4b4b27c365d2a386ef5107cd854/preview.webp" width="190" alt="base val-001"></a><br><sub>base val-001</sub> |
| 1 | 0 | `script` | **accepted** | yes | 0.6567 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/228a7fe756a8c04f2b7dc6d27c9a485d8eec263396087c8d909683d125afd3c9/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/228a7fe756a8c04f2b7dc6d27c9a485d8eec263396087c8d909683d125afd3c9/preview.webp" width="190" alt="script-v1 train-003"></a><br><sub>script-v1 train-003</sub> |
| 2 | 1 | `storyboard` | skipped | no | — | — |
| 3 | 1 | `director` | skipped | no | — | — |
| 4 | 0 | `storyboard` | **accepted** | yes | 0.5274 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/631d708ac3df6588b1e37f60b3541a534089d4bbb6e4b19ead030ff9c347b776/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/631d708ac3df6588b1e37f60b3541a534089d4bbb6e4b19ead030ff9c347b776/preview.webp" width="190" alt="train-001"></a><br><sub>train-001</sub> |
| 5 | 0 | `director` | skipped | no | — | — |
| 6 | 1 | `memory` | skipped | no | — | — |
| 7 | 0 | `memory` | **accepted** | yes | 0.6285 | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/408a9efa4063f15afcf790f0a0613a53d74c62d351793674f1cfe3b7e50f9fae/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/408a9efa4063f15afcf790f0a0613a53d74c62d351793674f1cfe3b7e50f9fae/preview.webp" width="190" alt="train-002"></a><br><sub>train-002</sub> |
| 8 | 1 | `script` | skipped | no | — | — |
| 9 | 1 | `storyboard` | skipped | no | — | — |
| 10 | 4 | `director` | **accepted** | yes | **0.6749** | <a href="https://github.com/HaFred/H3-rsi/blob/main/evaluations/b2f33fd8e5b9a326f9a066d57667edd5ee70bc854e10b84dca9ece6ec39e2850/video.mp4"><img src="https://github.com/HaFred/H3-rsi/raw/main/evaluations/b2f33fd8e5b9a326f9a066d57667edd5ee70bc854e10b84dca9ece6ec39e2850/preview.webp" width="190" alt="best val-001"></a><br><sub>best val-001</sub> |
| 11 | 4 | `memory` | skipped | no | — | — |
| 12 | 4 | `script` | skipped | no | — | — |
| 13 | 4 | `storyboard` | skipped | no | — | — |

- 13 reflection calls, 5 candidates kept, 62 metric calls, 45 H3 requests (the remainder were cache hits on
  already-scored cases). Search took 2317 s (≈39 min).
- Iterations 1, 4 and 7 passed the minibatch and then **lost** on full validation (0.6567, 0.5274, 0.6285 are all
  below the 0.6588 baseline) — the loop rejecting its own ideas, which is why the ledger matters as much as the
  final number.
- The winner came from iteration 10, where `director` was rewritten on top of the already-accepted `script`.
- `memory` was accepted twice on minibatches but never became the best program; the champion's `memory` is empty.

---

## 5. Validity guard in action

The base harness's val-002 attempt at 14:25 was produced by H3, then **rejected** with
`invalid reward signals: malformed video judge frames` and scored 0. The evaluator re-ran the case, the judge
returned usable frames, and it scored 0.674. No invalid evaluation was counted in the report, and the rejected
clip was not silently reused. This is why a bad clip can never quietly become a data point: it either gets a
valid score or the run stops as a service failure.

---

## 6. Reproduce

```bash
cd fred_design/RSI_1st_impl

# services: H3 fl2va on GPU 6,7 (port 30010) + policy 8000 / reflection 8001 / Jev 8085 / judge 8090 on GPU 5
source ../h3_serving/env_sglang.sh
CUDA_VISIBLE_DEVICES=6,7 VARIANT=fl2va PORT=30010 NUM_GPUS=2 ULYSSES_DEGREE=1 PERFORMANCE_MODE=memory \
  SGLANG_EXTRA_ARGS="--tp-size 2 --layerwise-offload-components dit,text_encoder,vae --layerwise-resident-layers video_vae=36 --dit-offload-prefetch-size 1 --dit-layerwise-resident-layers 20 --enable-torch-compile false" \
  bash scripts/serve_minimax_h3.sh

# the loop (run_id / budgets as pinned in run_config.resolved.yaml)
PYTHONPATH=. python -m rsi_harness.cli optimize --run-config configs/gepa_video_h800.yaml
PYTHONPATH=. python -m rsi_harness.cli promote  --run-dir outputs/101214_1strun
PYTHONPATH=. python -m rsi_harness.cli report   --run-dir outputs/101214_1strun --test
```

| Artifact | Contents |
| --- | --- |
| `readme.md` | auto-generated run summary |
| `base.yaml` / `best.yaml` | seed harness and the search winner |
| `search_report.json` | candidates, changed components, val scores, budget |
| `promotion/decision.json` | gate inputs and verdict (`accepted: true`) |
| `report.json` | search + promotion + test summary |
| `candidates/<hash>/harness.yaml` | full text of every candidate |
| `evaluations/<key>/video.mp4` + `preview.webp` / `preview-lg.webp` + `outcome.json` + `trajectory.json` | every clip (full rate + the inline animated preview), its score, the judge feedback and the prompts that made it |
| `events.jsonl` | time-ordered log of every evaluation, reflection call and decision |
| `gepa/run_log.txt` | raw GEPA iteration log |

## 7. Caveats

- One seed per task. Per-task deltas of ~0.01–0.02 are within judge noise; the promotion mean over 6 tasks and
  the test mean over 4 tasks are the meaningful numbers.
- The chain is **stochastic**, so a single clip pair is not a controlled A/B. Steps 1 and 2 share a byte-identical
  `prompts.script`, yet produced different scripts (851 vs 1,048 ch) — the text LLM samples at `temperature: 0.7`,
  and the three steps ran under three different harness hashes. Treat the per-task clips in §2 as illustrations of
  the mechanism, not as isolated evidence for one edit.
- The `story` axis moves against the `video` axis on some tasks (train-001 −0.06, test-001 +0.03). The gate only
  requires that no axis regress beyond 0.02 on the promotion mean, which held.
- The clips in section 2 are one task each. The claim they support is that the edits changed pacing and
  alignment in the expected direction, which the aggregate tables confirm; they are not proof that the champion
  is better on every task.
- The champion is `gepa-base.1`. The next run starts from it, so this loop is meant to repeat rather than to
  finish in one pass.
