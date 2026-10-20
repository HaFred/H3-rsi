# Source case data

These are the generation inputs for the single-clip and five-clip experiments, copied from `rsi-1st-impl-design/benchmarks/`. They are manually authored pilot tasks, not a large external dataset. Prompt text, identifiers, split assignments, and generation/continuity specifications are preserved.

| Dataset | Entry point | Independent tasks | Clips per task | Train / Validation / Promotion / Test |
|---|---|---:|---:|---|
| Single clip | [tasks_gepa_v1.jsonl](tasks_gepa_v1.jsonl) | 16 | 1 | 4 / 2 / 6 / 4 |
| Five clips | The 12 `longform_*_5clip_v1.yaml` files below | 12 projects | 5 | 2 / 4 / 4 / 2 |

## Single-clip inputs

Each JSONL row is a task with `task_id`, the original `prompt`, `task_type`, `split`, and `extra.source_id`. These are task requirements, distinct from reusable Harness system prompts and per-evaluation generated scripts/storyboards/director prompts.

### Single-clip field reference

[`tasks_gepa_v1.jsonl`](tasks_gepa_v1.jsonl) is JSON Lines: each line is one independent task object. The following fields occur in the published file.

| Field | Example | Role in the RSI loop |
| --- | --- | --- |
| `task_id` | `train-001` | Unique task-row identity. Combined with split and generation seed to form an evaluation case ID such as `train:train-001:seed0`; links results back to the input row. |
| `prompt` | A person in a yellow raincoat picks up a red umbrella... | Original generation requirement passed to the script agent. The frame-quality judge also uses this original text as its caption, rather than the director's derived prompt. RSI changes harness system prompts, not this case requirement. |
| `task_type` | `t2va` | Requested generation mode. The strict RSI controller requires it to match the frozen evaluation policy; all published cases use text-to-video/audio. |
| `split` | `train` | Assigns the task to train, validation, promotion or test. Train supplies reflection and parent/child comparisons; validation selects the search winner; promotion gates deployment; test reports results. The loader stores this value under runtime `extra.split`. |
| `extra.source_id` | `umbrella-01` | Stable source/provenance group identity, separate from the row's `task_id`. The strict controller rejects a source ID appearing in different splits, so variants of the same source must stay in one split. It is retained in evaluation case records and also used when checking imported experience for overlap with held-out sources. It is not a generation prompt or a reward component. |
| `extra.auto_route` | `false` | Disables the director's automatic generation-mode routing for these tasks, preserving the requested mode. The strict evaluator also reconstructs cases with routing disabled. |

For example, two differently named task rows derived from `umbrella-01` may share that source ID within train, but assigning one to train and the other to test makes the strict controller reject the dataset. If `source_id` is absent or empty, that split-overlap check falls back to `task_id`; therefore related variants need an explicit shared source ID. The published file contains 16 distinct source IDs. These are declared provenance groups, not automatic semantic-duplicate detection or paths to external assets.

The [single-clip manifest](../manifest.json) records these task IDs expanded with `seed0`, and its `identity.tasks_sha256` matches the published JSONL bytes. The Windows source checkout had CRLF line endings; this JSONL uses LF to recover the exact recorded run hash. No prompt content is changed.

## Five-clip inputs

Each YAML is a complete `ProjectSpec`: target settings, continuity contract/world, five segment prompts and dependencies, evidence/payoff checks, state commits, stitch order, and `split_policy.status`.

### ProjectSpec field reference

The table covers the entries used by the published `longform_*_5clip_v1.yaml` cases. These are fixed project inputs, separate from the harness components RSI searches. `segments[]` denotes each segment entry, not a separate input file.

| Field | Role in generation / evaluation |
| --- | --- |
| `schema_version` | Selects ProjectSpec schema version (`1`); unknown fields are rejected by the strict loader. |
| `project_id` | Stable identity for the whole project and its recorded artifacts. |
| `title` | Human-readable project name, included in generation briefs and feedback. |
| `kind` | Project category metadata, here `longform-continuity`. |
| `source_dataset` | Dataset provenance, here `original`. |
| `target.task_type` | Initial generation mode, `t2va`; the sequential producer uses `fl2va` with a prior-frame reference for later segments. |
| `target.short_edge`, `target.aspect_ratio` | Requested resolution and frame shape. |
| `target.duration_seconds`, `target.frame_rate` | Requested duration and fps for each segment, not the entire timeline. |
| `target.num_inference_steps` | H3 inference-step budget per generation. |
| `target.expected_frames` | Expected aligned frame count, checked against duration/fps with H3's `17n+5` alignment (124 here). |
| `continuity_contract.project_world` | Shared story/world context included in every segment brief. |
| `continuity_contract.recurring_motifs` | Declared recurring motif metadata; the current brief composer does not separately inject this list or turn it into automatic checks. Use prompts and explicit evidence/payoff questions to enforce intended appearances. |
| `continuity_contract.audio_policy`, `continuity_contract.visual_policy` | Shared instructions included in generation briefs; concrete stitching controls live under `stitch`. |
| `segments[].index`, `segments[].segment_id` | Consecutive 1-based position and unique segment identity; used for ordering, references and records. |
| `segments[].source_task_id` | Source-task provenance; does not assign the project's split. |
| `segments[].purpose` | Narrative function of the segment, included in its brief. |
| `segments[].prompt` | Original scene/action/audio requirement used to build the generation brief. |
| `segments[].depends_on` | Earlier segments that must be available; dependencies may only point backward. This is separate from the sequential last-frame handoff. |
| `segments[].state_commit` | Proposed fact key/value updates. Accepts a mapping or a list of single-key mappings; values must be strings. Updates enter StoryState only after matching evidence passes. |
| `segments[].evidence` | Fact-key -> visual yes/no question mapping; keys must exactly match `state_commit`. Confirmations determine downstream state and `state_confirmed_rate`. |
| `segments[].payoff` | Earlier fact-key -> visual yes/no question mapping, checking continuity in this segment. Keys must have been declared in earlier segments; runtime checks run only for already confirmed keys. Results affect `payoff_rate`, not state. |
| `stitch.order` | Playback/assembly sequence; must equal segment IDs in file order. |
| `stitch.video_transition` | Video transition, currently `cut`. |
| `stitch.audio_crossfade_seconds` | Audio overlap at segment boundaries, 0.12 s in these cases. |
| `stitch.output` | Output timeline filename, normally `timeline.mp4`. |
| `split_policy.status` | Project-level assignment: train, validation, promotion or test. Controls the case's role in search and independent evaluation. |
| `split_policy.rule` | Human-readable split/provenance rationale; does not itself implement a new splitting algorithm. |

The schema also supports optional `stitch.loudness_target_lufs` (default -23 LUFS), omitted in these cases. Evidence/payoff answers use the maximum P(yes) over sampled middle/end frames with a 0.5 confirmation threshold. An answer of "no" alone does not trigger a retry. See the [RSI question-flow explanation](../README.md#23-five-clip-visual-evidence-and-its-video-mapping) for state, scoring and reflection effects.

| Source YAML | Project ID | Split | Clips |
|---|---|---|---:|
| [longform_lighthouse_storm_5clip_v1.yaml](longform_lighthouse_storm_5clip_v1.yaml) | `lighthouse-storm-5clip-v1` | train | 5 |
| [longform_town_fair_5clip_v1.yaml](longform_town_fair_5clip_v1.yaml) | `town-fair-5clip-v1` | train | 5 |
| [longform_mountain_bus_5clip_v1.yaml](longform_mountain_bus_5clip_v1.yaml) | `mountain-bus-5clip-v1` | validation | 5 |
| [longform_night_market_close_5clip_v1.yaml](longform_night_market_close_5clip_v1.yaml) | `night-market-close-5clip-v1` | validation | 5 |
| [longform_rooftop_garden_5clip_v1.yaml](longform_rooftop_garden_5clip_v1.yaml) | `rooftop-garden-5clip-v1` | validation | 5 |
| [longform_snow_postman_5clip_v1.yaml](longform_snow_postman_5clip_v1.yaml) | `snow-postman-5clip-v1` | validation | 5 |
| [longform_desert_caravan_5clip_v1.yaml](longform_desert_caravan_5clip_v1.yaml) | `desert-caravan-5clip-v1` | promotion | 5 |
| [longform_harbor_dawn_5clip_v1.yaml](longform_harbor_dawn_5clip_v1.yaml) | `harbor-dawn-5clip-v1` | promotion | 5 |
| [longform_science_fair_5clip_v1.yaml](longform_science_fair_5clip_v1.yaml) | `science-fair-5clip-v1` | promotion | 5 |
| [longform_tea_farm_5clip_v1.yaml](longform_tea_farm_5clip_v1.yaml) | `tea-farm-5clip-v1` | promotion | 5 |
| [longform_autumn_orchard_5clip_v1.yaml](longform_autumn_orchard_5clip_v1.yaml) | `autumn-orchard-5clip-v1` | test | 5 |
| [longform_subway_musician_5clip_v1.yaml](longform_subway_musician_5clip_v1.yaml) | `subway-musician-5clip-v1` | test | 5 |

One long-form case is one project × one generation seed for one Harness. Increasing the number of seeds increases cases, not independent projects. These project splits are separate from the single-clip JSONL splits. Segment `source_task_id` values record provenance; they do not assign the long-form project to that short-clip split or make the two datasets mutually independent. Long-form results must not be reused as single-clip held-out results.

The same 12 YAML files were already published under [longform/101704_1strun/projects/](../longform/101704_1strun/projects/). This directory adds a common source-data entry point matching the implementation's `benchmarks/` paths. Every YAML here has the same content as that archived copy and matches its recorded hash in the [long-form manifest](../longform/101704_1strun/manifest.json). Windows checkout CRLF endings are normalized to LF to recover the exact recorded run bytes. Keep the run archive as the provenance record for that run.

## Integrity and use

[checksums.json](checksums.json) records SHA-256 for all 13 source files, source paths, and the archived long-form locations. Local Git attributes preserve the snapshot bytes. The original YAML language is retained; these source files are not translated slide summaries.

Train supports reflection and parent/child selection; Validation selects the search winner; Promotion independently accepts or rejects it; Test reports the selected versions without feeding back into search. Search and evaluation policy, model checkpoints, seeds, and budgets remain separate run configuration; these files alone are not the complete reproduction environment.
