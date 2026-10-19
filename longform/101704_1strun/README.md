# Longform run `101704_1strun`: archived evidence and figure provenance

This archive accompanies the [longform section of the repository README](../../README.md#longform-rsi-five-clips-searched-as-one-project).
The run completed; its search winner was **rejected** by the independent promotion gate.
All MP4 locations below are recorded server-side identities. No MP4s or dangling video links are included.

## Frozen artifacts

| Artifact | Purpose |
| --- | --- |
| [manifest.json](manifest.json) | Run config, service revisions, source commit, project hashes and split definitions |
| [base.yaml](base.yaml) / [best.yaml](best.yaml) | Complete execution harness snapshots; the winner changes only memory |
| [search.json](search.json) | 44 evaluated configurations; full validation scores for base and 3 child candidates |
| [report.json](report.json) | Search, independent promotion and final test results |
| [promotion/decision.json](promotion/decision.json) | `accepted: false`; mean gain below 0.02 and tea-farm seed 0 regression beyond 0.05 |
| [calibration.json](calibration.json) | Frozen search/report score policies; loop penalty disabled |
| [budget.json](budget.json) | Actual H3 request accounting: 950 search + 80 promotion + 20 test |
| [events.jsonl](events.jsonl) | Evaluation and lifecycle ledger |
| [gepa/run_log.txt](gepa/run_log.txt) / [gepa/candidates.json](gepa/candidates.json) | Search trace and GEPA candidate texts |
| [projects/](projects/) | 12 original ProjectSpecs; every file matches the SHA-256 in the manifest |
| [videos.json](videos.json) | Complete recorded case/harness/project-directory/timeline mapping, including failed attempts |
| [figure_video_map.json](figure_video_map.json) | Explicit machine-readable mapping for all six original WebPs |
| [previews/](previews/) | Animated WebP previews derived from the recorded timelines and stitched segments, plus [provenance.json](previews/provenance.json) |
| [checksums.json](checksums.json) | SHA-256 of copied source artifacts, including the untouched WebPs |

`S_project` is the search objective (0.70 segment quality + 0.30 handoff).
`R_project` is the report/gate objective (0.40 segment quality + 0.20 confirmed state + 0.20 payoff + 0.20 handoff,
with missing-metric renormalization). `C = mean(R_clip)` is the segment-quality component, not the project report score.
Some original figure captions predate this notation; the following guide makes the distinction explicit.

## Figure guide

| Original WebP | What it represents | Interpretation / known caption issue |
| --- | --- | --- |
| [Search](figures/101704_1strun_search.webp) | Validation `S_project` of the 3 child candidates; orange reference is the base | Read alongside `search.json`; the base is a fourth full-validation configuration |
| [Five-segment sheet](figures/101704_1strun_base_vs_best.webp) | Last frame of each segment of desert-caravan, promotion seed 1; top base, bottom winner | Legacy `S` means `R_project`; columns are alphabetic, not chronological |
| [Evidence frames](figures/101704_1strun_evidence.webp) | Three segment checks, with base/winner frame pairs | Probabilities are annotations from the original figure; no judge was queried again for this publication |
| [Promotion](figures/101704_1strun_promotion.webp) | Base/winner `R_project` for all 8 promotion project×seed cases | Legacy `S` means `R_project`; the legend's `script + storyboard` is stale: the winner changes **memory only** |
| [Metric decomposition](figures/101704_1strun_axes.webp) | Promotion/test means of project score, segment score, axes and continuity diagnostics | `S (project score)` means `R_project`; `R = segment_score_mean` means `C = mean(R_clip)` |
| [Timeline](figures/101704_1strun_timeline.webp) | Approximation of cumulative requests using `5 × project_done` events | Use `budget.json` for actual requests: incomplete projects can use fewer than five. The footer about three crash/resume sessions is stale and unsupported by this run's event ledger |

All six WebPs are byte-for-byte copies. Their captions were not retouched. The JSON tables in the main README
use the frozen reports directly and take precedence over legacy figure labels.

## Exact video mapping for the five-segment sheet

Case: `promotion:desert-caravan-5clip-v1:seed1`. Top is base `531468a18cbe…`; bottom is winner `1d079e4c4820…`.
The values in the original image, 0.775 and 0.867, are rounded **project report scores**:
`0.77452 → 0.86662`, a gain of `0.09210`.

| Playback index | Segment | WebP column | Displayed source frame |
| --- | --- | --- | --- |
| 1 | dune-walk | 1 | Last frame, index 123 |
| 2 | sandstorm | 3 | Last frame, index 123 |
| 3 | oasis | 2 | Last frame, index 123 |
| 4 | tent-tea | 5 | Last frame, index 123 |
| 5 | starry-night | 4 | Last frame, index 123 |

This ordering comes from [the frozen stitch order](projects/longform_desert_caravan_5clip_v1.yaml).
The source figure generator sorted `references/*.last.png` by filename, which explains its different column order.

Every hash below is a directory under the recorded server path `outputs/longform_rsi/101704_1strun/projects/`;
the stitched movie in each is named `timeline.mp4`.

| Case | Arm | Recorded project directory hash |
| --- | --- | --- |
| `promotion:desert-caravan-5clip-v1:seed1` | base | `cae591d6210f6ac740d13a7d4b2448b6cc6cac1d6c3db33c558c7fdcb95324fc` |
| `promotion:desert-caravan-5clip-v1:seed1` | best | `5b86a07a8cecb7a7697120f8888c23635e8d4b7cb77d22c0d83c3ce560d06aa1` |
| `promotion:desert-caravan-5clip-v1:seed0` | base | `a8e54822a20937ea7adfc7151042c80b39d22f709783dfd027e43489d8af90cc` |
| `promotion:desert-caravan-5clip-v1:seed0` | best | `eff961c9948e5fd930408e733f9e0af2ae0b3dc01b052aa462346f36ce28271c` |
| `promotion:harbor-dawn-5clip-v1:seed0` | base | `7186859fd0311f76dfc65d759bacefa46206054f300a2dcd9a5b6da11557dc8c` |
| `promotion:harbor-dawn-5clip-v1:seed0` | best | `520e6420592d9344b8824123f8a4795b48b75e5faad4fd32d72c3e0bdc5e89b2` |

The same entries in `videos.json` give their recorded `videos/base/...mp4` and `videos/best/...mp4` links.
They identify provenance only; they do not resolve inside this archive.

## Exact video mapping for the evidence figure

The displayed columns are source-frame indices **62 and 123** of the identified segment (124 frames total).
Within each block, base is above winner. The blocks, top to bottom, are:

1. `promotion:desert-caravan-5clip-v1:seed0`, segment `tent-tea`, evidence question `tea`.
2. `promotion:desert-caravan-5clip-v1:seed1`, segment `sandstorm`, payoff question `camels`.
3. `promotion:harbor-dawn-5clip-v1:seed0`, segment `sun-up`, evidence question `time`.

The sampled frames come from the corresponding segment clip under
`<project_dir>/segments/<index>_<segment_id>/attempt_1/`, rather than from independently selected timelines.
The recorded probability takes the maximum over the two sampled frames; it is not their mean or a whole-video probability.
The original figure's claim about re-querying within ~0.02 is retained as source-image text; no new repeatability claim is made here.

These mappings were reconstructed from the original figure labels, the generator's selection rules, the
frozen `videos.json` and the hashed ProjectSpecs. The MP4s were unavailable locally and were not re-decoded.
