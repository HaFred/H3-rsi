# Source case data

These are the generation inputs for the single-clip and five-clip experiments, copied from `rsi-1st-impl-design/benchmarks/`. They are manually authored pilot tasks, not a large external dataset. Prompt text, identifiers, split assignments, and generation/continuity specifications are preserved.

| Dataset | Entry point | Independent tasks | Clips per task | Train / Validation / Promotion / Test |
|---|---|---:|---:|---|
| Single clip | [tasks_gepa_v1.jsonl](tasks_gepa_v1.jsonl) | 16 | 1 | 4 / 2 / 6 / 4 |
| Five clips | The 12 `longform_*_5clip_v1.yaml` files below | 12 projects | 5 | 2 / 4 / 4 / 2 |

## Single-clip inputs

Each JSONL row is a task with `task_id`, the original `prompt`, `task_type`, `split`, and `extra.source_id`. These are task requirements, distinct from reusable Harness system prompts and per-evaluation generated scripts/storyboards/director prompts.

The [single-clip manifest](../manifest.json) records these task IDs expanded with `seed0`, and its `identity.tasks_sha256` matches the published JSONL bytes. The Windows source checkout had CRLF line endings; this JSONL uses LF to recover the exact recorded run hash. No prompt content is changed.

## Five-clip inputs

Each YAML is a complete `ProjectSpec`: target settings, continuity contract/world, five segment prompts and dependencies, evidence/payoff checks, state commits, stitch order, and `split_policy.status`.

| Source YAML | Project ID | Split | Clips |
|---|---|---|---:|
| [longform_autumn_orchard_5clip_v1.yaml](longform_autumn_orchard_5clip_v1.yaml) | `autumn-orchard-5clip-v1` | test | 5 |
| [longform_desert_caravan_5clip_v1.yaml](longform_desert_caravan_5clip_v1.yaml) | `desert-caravan-5clip-v1` | promotion | 5 |
| [longform_harbor_dawn_5clip_v1.yaml](longform_harbor_dawn_5clip_v1.yaml) | `harbor-dawn-5clip-v1` | promotion | 5 |
| [longform_lighthouse_storm_5clip_v1.yaml](longform_lighthouse_storm_5clip_v1.yaml) | `lighthouse-storm-5clip-v1` | train | 5 |
| [longform_mountain_bus_5clip_v1.yaml](longform_mountain_bus_5clip_v1.yaml) | `mountain-bus-5clip-v1` | validation | 5 |
| [longform_night_market_close_5clip_v1.yaml](longform_night_market_close_5clip_v1.yaml) | `night-market-close-5clip-v1` | validation | 5 |
| [longform_rooftop_garden_5clip_v1.yaml](longform_rooftop_garden_5clip_v1.yaml) | `rooftop-garden-5clip-v1` | validation | 5 |
| [longform_science_fair_5clip_v1.yaml](longform_science_fair_5clip_v1.yaml) | `science-fair-5clip-v1` | promotion | 5 |
| [longform_snow_postman_5clip_v1.yaml](longform_snow_postman_5clip_v1.yaml) | `snow-postman-5clip-v1` | validation | 5 |
| [longform_subway_musician_5clip_v1.yaml](longform_subway_musician_5clip_v1.yaml) | `subway-musician-5clip-v1` | test | 5 |
| [longform_tea_farm_5clip_v1.yaml](longform_tea_farm_5clip_v1.yaml) | `tea-farm-5clip-v1` | promotion | 5 |
| [longform_town_fair_5clip_v1.yaml](longform_town_fair_5clip_v1.yaml) | `town-fair-5clip-v1` | train | 5 |

One long-form case is one project × one generation seed for one Harness. Increasing the number of seeds increases cases, not independent projects. These project splits are separate from the single-clip JSONL splits. Segment `source_task_id` values record provenance; they do not assign the long-form project to that short-clip split or make the two datasets mutually independent. Long-form results must not be reused as single-clip held-out results.

The same 12 YAML files were already published under [longform/101704_1strun/projects/](../longform/101704_1strun/projects/). This directory adds a common source-data entry point matching the implementation's `benchmarks/` paths. Every YAML here has the same content as that archived copy and matches its recorded hash in the [long-form manifest](../longform/101704_1strun/manifest.json). Windows checkout CRLF endings are normalized to LF to recover the exact recorded run bytes. Keep the run archive as the provenance record for that run.

## Integrity and use

[checksums.json](checksums.json) records SHA-256 for all 13 source files, source paths, and the archived long-form locations. Local Git attributes preserve the snapshot bytes. The original YAML language is retained; these source files are not translated slide summaries.

Train supports reflection and parent/child selection; Validation selects the search winner; Promotion independently accepts or rejects it; Test reports the selected versions without feeding back into search. Search and evaluation policy, model checkpoints, seeds, and budgets remain separate run configuration; these files alone are not the complete reproduction environment.
