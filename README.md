# RSI × MiniMax-H3 — single-clip and five-clip evidence

This repository records two real H3 experiments. [Longform run `101416_1strun`](#2-longform-rsi-five-clips-searched-as-one-project)
searches a harness by generating and scoring complete five-clip projects. [Single-clip run `101214_1strun`](#3-single-clip-rsi-101214_1strun)
is the earlier promoted demo. Their scores use different evaluation units and policies; compare base and candidate within each run.

**Metric notation:** `R_clip` is one clip reward and `S_clip` its search objective; `S_project` and `R_project` score complete projects, sharing `C = mean(R_clip)`. These names follow the implementation documentation. See the [score reference](slides_md/07-data-splits-and-scores.md) for formulas and split usage.

**Contents**

- [0. Design Walkthrough](slides_md/README.md)
- [1. Source case data](#1-source-case-data)
- [2. Longform RSI: five clips searched as one project](#2-longform-rsi-five-clips-searched-as-one-project)
  - [2.1. What one evaluation generates](#21-what-one-evaluation-generates)
  - [2.2. Search and independent results](#22-search-and-independent-results)
  - [2.3. Five-clip visual evidence and its video mapping](#23-five-clip-visual-evidence-and-its-video-mapping)
  - [2.4. Cost, scope and archived evidence](#24-cost-scope-and-archived-evidence)
- [3. Single-clip RSI: 101214_1strun](#3-single-clip-rsi-101214_1strun)

## 1. Source case data

The original generation requirements are available in [benchmarks/](benchmarks/README.md):

- [Single-clip tasks](benchmarks/tasks_gepa_v1.jsonl): 16 task prompts, split 4/2/6/4 across Train/Validation/Promotion/Test.
- [Five-clip ProjectSpecs](benchmarks/README.md#five-clip-inputs): 12 YAML projects, each with five segment prompts, continuity rules, and stitch order, split 2/4/4/2.
- [SHA-256 manifest](benchmarks/checksums.json): links the published sources to the recorded run inputs.

## 2. Longform RSI: five clips searched as one project

Run **`101416_1strun`** searches a harness by generating complete five-clip projects. Its final winner, **`c1912a62`**, improves mean Validation search score and **passes the independent promotion gate**. The champion store nevertheless stays unchanged because its current champion does not match this run's expected base. The town-fair videos below illustrate the same run's recursive rewrite lineage, with remaining failures visible alongside the score gains.

**Net run duration: 12 h 33 min** of recorded active-session wall time (**45,172 s**, lower bound) across **four processes**, with **25 h 36 min** of downtime between sessions excluded. The report's 20 min 56 s covers only the final cache-replay session. [Session table, calculation and timing evidence](slides_md/subdocs_examples/README.md#how-search-and-promotion-relate).

### 2.1. What one evaluation generates

One case is **one five-segment project × one seed**, generated with one candidate harness throughout. The first segment uses H3 `t2va`; segments 2–5 use `fl2va`, conditioned on the preceding segment's last frame. Visually confirmed facts enter a persistent per-project `StoryState` and constrain later segments. Once all five segments are committed, FFmpeg stitches them into `timeline.mp4`. Each requested 5 s segment has 124 frames at 24 fps; five segments give **620 frames, about 25.83 s**.

```text
candidate harness + project brief
  → clip 1 (t2va)
  → last frame + confirmed state → clip 2 (fl2va)
  → last frame + confirmed state → clip 3 (fl2va)
  → last frame + confirmed state → clip 4 (fl2va)
  → last frame + confirmed state → clip 5 (fl2va)
  → stitched timeline + project metrics
  → Train feedback → GEPA proposes another harness candidate
```

GEPA compares complete sequential projects, rather than picking five independent clip winners and concatenating them. The project score aggregates segment quality, confirmed state, payoff checks and boundary similarity; it is not a single judge grading the whole stitched movie.

H3 and the served production/reflection/judge models stay frozen. The run starts from [`gepa-base`, candidate `d16c2e95`](longform/101416_1strun/base.yaml). The selected [`c1912a62` harness](longform/101416_1strun/best.yaml) changes **Script and Storyboard system prompts** relative to that base. Director, Critic and Editor prompts remain unchanged, and shared memory is empty in both snapshots. The [worked town-fair comparison](slides_md/subdocs_examples/11-town-fair-longform-production.md) expands the base and first Script rewrite `c863aae2` into their actual agent outputs, H3 requests, state transitions and stitching.

### 2.2. Search and independent results

Two project scores serve different purposes:

- **`S_project = 0.70 C + 0.30 H`** drives Train acceptance, Validation ranking and winner selection.
- **`R_project = 0.40 C + 0.20 F + 0.20 P + 0.20 H`** is the project reward/report score and the independent promotion metric when all components are present.

`C = mean(R_clip)`, with `R_clip = 0.60 video + 0.25 story + 0.15 editing`; `F` is the confirmed-state rate, `P` the pass rate over eligible payoff checks actually executed, and `H` the mean similarity of adjacent last/first frames. The frozen [policy](longform/101416_1strun/calibration.json) renormalizes missing metrics and disables the loop penalty. Both S_project and R_project can be measured on search cases; Promotion measures new R_project values on held-out cases. See the [score reference](slides_md/07-data-splits-and-scores.md).

| Split | Evaluation cases per harness | Metric | Base `d16c2e95` | Winner `c1912a62` | Δ |
| --- | --- | --- | ---: | ---: | ---: |
| Validation | 4 projects × seed 0 | `S_project` | 0.729348 | 0.751625 | +0.022277 |
| Validation | same 4 cases | `R_project` | 0.764495 | 0.767205 | +0.002710 |
| Promotion | 4 projects × seeds 0,1 = 8 cases | `R_project` | 0.792667 | 0.824728 | +0.032061 |
| Test, report only | 2 projects × seed 0 | `R_project` | 0.715060 | 0.722860 | +0.007800 |

Sources: recovered [`search.json`](longform/101416_1strun/search.json), [`report.json`](longform/101416_1strun/report.json), [`promotion/decision.json`](longform/101416_1strun/promotion/decision.json) and [`videos.json`](longform/101416_1strun/videos.json). Validation R_project means are calculated from the four indexed projects' C/F/P/H under the frozen policy. The gate records `mean_delta = 0.03206`; subtracting the displayed rounded Promotion means gives 0.032061.

The recovered search archive contains **22 distinct candidates**, including the base; **9** have full Validation scores. Train uses two projects × seeds 0,1, so each full parent/child comparison spans four project cases. Train feedback guides reflection, Validation selects the winner, and Promotion/Test do not supply reflection feedback.

The displayed town-fair lineage is a subset of this search:

| Harness | Rewrite in the lineage | Mean Validation S_project | Mean Validation R_project |
| --- | --- | ---: | ---: |
| `d16c2e95` | Base | 0.729348 | 0.764495 |
| `c863aae2` | Round 1: Script | 0.743650 | 0.769800 |
| `3252791a` | Round 7: Storyboard | 0.747713 | 0.783390 |
| `c1912a62` | Round 57: Script again; selected winner | 0.751625 | 0.767205 |

The final edit increases S_project while reducing R_project relative to `3252791a`. Winner selection uses S_project, so the search winner need not have the highest Validation R_project.

**Promotion required** mean R_project gain ≥ 0.02, no project×seed regression beyond 0.05, and no loss of validity. The winner passes: mean gain **+0.03206**, worst case **−0.02498**, with all eight cases valid in both arms. All final Test timelines are also valid.

**Gate accepted; champion store unchanged.** The saved store note says the current champion was `gepa-base.1`, whereas this run's expected base was `gepa-base`. The accepted verdict was recorded, but the store update was not applied. Gate acceptance must not be reported as a successful champion replacement.

<details>
<summary>Winner: Evaluated with Promotion/Test case, using R_project</summary>

The per-case results show that the winner improves six of eight Promotion cases, keeps both regressions within the allowed 0.05, and improves both report-only Test cases, supporting gate acceptance without implying a win on every project.

| Split | Project | Seed | Base | Winner | Δ |
| --- | --- | --- | ---: | ---: | ---: |
| Promotion | desert-caravan | 0 | 0.81588 | 0.81996 | +0.00408 |
| Promotion | desert-caravan | 1 | 0.81374 | 0.86180 | +0.04806 |
| Promotion | harbor-dawn | 0 | 0.84378 | 0.82008 | -0.02370 |
| Promotion | harbor-dawn | 1 | 0.73784 | 0.86494 | +0.12710 |
| Promotion | science-fair | 0 | 0.74774 | 0.82600 | +0.07826 |
| Promotion | science-fair | 1 | 0.73876 | 0.75238 | +0.01362 |
| Promotion | tea-farm | 0 | 0.82538 | 0.85942 | +0.03404 |
| Promotion | tea-farm | 1 | 0.81822 | 0.79324 | -0.02498 |
| Test | autumn-orchard | 0 | 0.75014 | 0.75182 | +0.00168 |
| Test | subway-musician | 0 | 0.67998 | 0.69390 | +0.01392 |

</details>

### 2.3. Five-clip visual evidence and its video mapping

These **animated WebP previews** play the complete five-segment [town-fair production](benchmarks/longform_town_fair_5clip_v1.yaml), about 25.83 s each. Click a preview for the source MP4 with sound. All four harnesses belong to **`101416_1strun`**, the same search summarized above; these particular videos are **Train productions**, not the held-out Promotion movies.

The four rows are single Train productions, not held-out cases, and they do not all share one seed: base `d16c2e95`, storyboard `3252791a` and final winner `c1912a62` are the Train **seed 1** production, while the first Script rewrite `c863aae2` is Train **seed 0**. All four identities were resolved from bytes, not appearance: each MP4 is byte-identical to exactly one of the index's 149 stitched timelines, and the other seed of the same candidate hashes differently. See [§8.1](slides_md/subdocs_examples/11-town-fair-longform-production.md#81-resolve-the-two-original-production-identities) and [§8.2](slides_md/subdocs_examples/11-town-fair-longform-production.md#82-storyboard-and-final-winner-identities) of the worked example. Base, storyboard and final winner form a same-seed lineage, and the storyboard-versus-winner pair is directly comparable; the Script rewrite differs in seed as well as in prompt.

<details>
<summary>Source identity of each preview: case, seed, project directory and byte proof</summary>

| Stage | Harness | Case id | Seed | Project directory | `sha256(timeline.mp4)` = `sha256(mediaN.mp4)` |
| --- | --- | --- | ---: | --- | --- |
| Base | `d16c2e95` | `train:town-fair-5clip-v1:seed1` | 1 | `e51364b4275d90a511b9599c4827e8ec6a7c09cda5804d81853712d1a561386b` | `629240f030c92c314e700d874f39f976805b3ade93caaa79eccd38c8fa5917a7` |
| Script rewrite | `c863aae2` | `train:town-fair-5clip-v1:seed0` | 0 | `51e22c0d5bb51b4d9a29ba3d6f01054e4d405a4ceb52c83e7d1ef0abfb49d96d` | `9905483d9076add66ee1f8bbf980416700eccf2f76c874a94ec59e10f436f99d` |
| Storyboard rewrite | `3252791a` | `train:town-fair-5clip-v1:seed1` | 1 | `e17b3a0dc1ceeb5e722a67ba0a36eb066123aaa742959d48b5833b259abb4585` | `561588f7ac43b59101a1a175843511d3d7a71dbc7a2f47d08f9b8668ac342c3c` |
| Script rewrite again | `c1912a62` | `train:town-fair-5clip-v1:seed1` | 1 | `b068f9914c4e79d03497ebb4c6bbb96fe13a29ac15232ce0d6f4e9347f2ea667` | `4637eb7034927e08fab07e03efbe66b8ea6588255cb517ad8dfa6ec4318e8700` |

Rejected alternatives under `outputs/longform_rsi/101416_1strun/projects/`: base seed 0 `0a3c683f…` hashes to `8159671e…f1941`; script rewrite seed 1 `37320791…` to `6a9dd098…18d6`; storyboard seed 0 `bbb388da…` to `16456132…4247e8c5`; final winner seed 0 `634dd89b…` to `dd2e52c1…42da2d6c`. Each matched `run.json` records the full candidate hash, the case id and seed, and the code and policy hashes; `videos.json` lists the same rows as `valid: true`. As an independent check that does not use the video bytes, all four seed-0 projects have a per-segment `harness_hash` equal to their project-level `artifact.harness_hash`, and all four seed-1 projects have a per-segment hash that differs (`6840390c…`, `7856b9f7…`, `09417f66…`, `0e7800c1…`), because `policy.apply(config, seed=1)` alters the hashed config.

</details>

Previews are 640 px wide at 8 fps, without audio or overlaid labels; [preview provenance](slides_md/previews/provenance.json) records the encoding and media hashes. Segments play in order: `umbrella-arrival` → `kite-park` → `fountain-arrival` → `bakery-prep` → `potter-opening`. The [world setting and segment requirements](slides_md/17-findings-and-pilot-prompts.md#town-fair-manually-constructed-five-segment-production-pilot) explain the intended actions. **Scores beside the four previews are means over four Validation projects, not scores for these town-fair videos.**

**Base** - `d16c2e95`, Train seed 1 preview; mean Validation S_project 0.7293 / R_project 0.7645

<a href="slides_md/assets/media4.mp4"><img src="slides_md/previews/town-fair_d16c2e95_base.webp" width="640" alt="Base harness d16c2e95: complete five-segment town-fair timeline"></a>

**1. Script rewrite** - `c863aae2` (round 1), Train seed 0 preview; mean Validation S_project 0.7436 / R_project 0.7698

<a href="slides_md/assets/media5.mp4"><img src="slides_md/previews/town-fair_c863aae2_script.webp" width="640" alt="Script rewrite c863aae2: complete five-segment town-fair timeline"></a>

**2. Storyboard rewrite** - `3252791a` (round 7), Train seed 1 preview; mean Validation S_project 0.7477 / R_project 0.7834

<a href="slides_md/assets/media6.mp4"><img src="slides_md/previews/town-fair_3252791a_storyboard.webp" width="640" alt="Storyboard rewrite 3252791a: complete five-segment town-fair timeline"></a>

**3. Script rewrite again** - `c1912a62` (round 57), Train seed 1 preview; mean Validation S_project 0.7516 / R_project 0.7672

<a href="slides_md/assets/media7.mp4"><img src="slides_md/previews/town-fair_c1912a62_script-again.webp" width="640" alt="Further script rewrite c1912a62: complete five-segment town-fair timeline"></a>

Each preview maps to one complete project, rather than independently selected clips. `c1912a62` is the search winner by mean Validation S_project; `3252791a` has the highest Validation R_project among these four. The independent gate result above concerns the final `c1912a62` winner on held-out projects, not any intermediate rewrite's Train video.

| Stage | Harness SHA prefix | Train seed | Source MP4 |
| --- | --- | ---: | --- |
| Base | `d16c2e95` | 1 | [media4.mp4](slides_md/assets/media4.mp4) |
| Script rewrite | `c863aae2` | 0 | [media5.mp4](slides_md/assets/media5.mp4) |
| Storyboard rewrite | `3252791a` | 1 | [media6.mp4](slides_md/assets/media6.mp4) |
| Script rewrite again | `c1912a62` | 1 | [media7.mp4](slides_md/assets/media7.mp4) |

<details>
<summary>Selected action, fixed visual checks and the detailed production comparison</summary>

The `kite-park` requirement asks for a child in a green jacket catching a falling paper kite. The comparison below shows the base and final winner's complete second segments, not selected stills. The winner shows a colorful kite in the child's hands, while the base has no clearly visible kite there. This supports inspecting the requested prop/action, but does not by itself prove the full catching motion or establish a judge probability.

| Requested action | Base `d16c2e95` | Winner `c1912a62` |
| --- | --- | --- |
| Child holding a kite, `kite-park` | <a href="slides_md/assets/media4.mp4#t=5.166667,10.333333"><img src="slides_md/previews/town-fair_d16c2e95_kite-park.webp" width="330" alt="Base town-fair kite-park: no clearly visible kite in the child's hands"></a> | <a href="slides_md/assets/media7.mp4#t=5.166667,10.333333"><img src="slides_md/previews/town-fair_c1912a62_kite-park.webp" width="330" alt="Final winner town-fair kite-park: child holding a colorful kite"></a> |

The [worked comparison](slides_md/subdocs_examples/11-town-fair-longform-production.md#5-expanded-segment-by-segment-comparison) covers **base versus the first Script rewrite `c863aae2`**, not the final winner. For those displayed productions, S_project is **0.72352 → 0.72819** and R_project **0.65242 → 0.80776**; confirmed updates rise **5/11 → 9/11**, and executed payoffs **3/5 → 6/6**. These are individual Train production scores at different seeds. They cannot substitute for Validation means or held-out gate scores.

| Fixed ProjectSpec field | Town-fair example | Role in this run |
| --- | --- | --- |
| `prompt` + `state_commit` | Segment 2 requests the child/kite action and proposes `kite: recovered` | Requirements drive generation; a declaration is not visual proof. |
| `evidence.<fact>` | `kite`: “Is a child in a green jacket holding a kite?” | At P(yes) ≥ 0.5 the proposed fact enters confirmed state; otherwise it is withheld and an older confirmed value for that key is removed. |
| `payoff.<fact>` | Segment 3 checks whether the earlier red umbrella remains visible | Runs only when the key is in upstream confirmed state; pass/fail contributes to P without updating state. Skipped checks are excluded from its denominator. |
| Project evaluation | F summarizes confirmed updates; P summarizes executed payoff checks | Both have weight 0.20 in R_project. Neither directly enters S_project, though changed state can affect later generation. |
| Train feedback → reflection | Weak segment scores, evidence and transition diagnostics | Guides harness changes. Questions stay fixed; Validation selects and held-out Promotion gates the winner. |

For the recovered base/first-rewrite pair, evidence/payoff judging samples the middle and last frames and takes the maximum P(yes), not a whole-video probability. A failed check alone does not invalidate or regenerate a segment. Passing a prop-presence check also does not prove continuous visibility, exact identity or successful motion. See the [ProjectSpec reference](benchmarks/README.md#projectspec-field-reference).

</details>

### 2.4. Cost, scope and archived evidence

The recovered [budget ledger](longform/101416_1strun/budget.json) records **716 actual H3 generation requests**: Search **615/1,000**, Promotion **80/120**, Test **21/50**. These count attempted segment generations, including failed/incomplete attempts; they are not counts of completed movies or text/judge calls. The run was interrupted and resumed; the final report restores search from cached results. It did not consume its entire Search budget.

The score has useful but limited signals: story/editing assess generated text plans, H can be high because the next clip is conditioned on the previous last frame, and payoff checks can be skipped when upstream facts were not confirmed. The recovered town-fair pair still shows later bakery/pottery scenes drifting toward the cyclist/fountain composition, fails the bowl check in both arms, and has a worse maximum audio jump in the rewrite (**14.24 → 33.98 dB**). Higher project scores do not establish uniformly better storytelling or sound.

The [101416 archive](longform/101416_1strun/README.md) contains byte-identical copies of the frozen base/winner, scoring policy, budget, search/report records, gate decision and video index, with [source hashes](longform/101416_1strun/provenance.json). The presentation MP4s and WebP previews are linked in section 2.3; the [worked example](slides_md/subdocs_examples/11-town-fair-longform-production.md) preserves recovered segment traces and the media identity proof for the first two rows, with [§8.2](slides_md/subdocs_examples/11-town-fair-longform-production.md#82-storyboard-and-final-winner-identities) resolving the other two. The earlier presentation snapshot described an interrupted run; the recovered final report and gate record establish the completed outcome summarized here. Active-session duration is recovered in section 2 (12 h 33 min across four processes), and every displayed video now has a recorded case, seed and project directory.

---

## 3. Single-clip RSI: 101214_1strun

The intro to `101214_1strun` is essentially recorded in [rfc748](https://github.com/verl-project/verl-omni/issues/748), here we elaborate on it with more details.
See [here](rfc748/readme.md).
