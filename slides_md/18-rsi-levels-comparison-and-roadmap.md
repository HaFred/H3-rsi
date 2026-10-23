# 18. Video RSI Levels, L3/L4 Comparisons, and the H3-rsi Roadmap

**Analysis supplement added on 2026-10-23; not part of the original 17-slide presentation.** Based on local `rsi-1st-impl-design/rsi_harness` source code, H3-rsi experiment records, and the linked papers. See the [full research analysis](../rfc748/H3_RSI_2026_Research_and_Conversation_Full.md) for supporting detail.

**Current position: H3-rsi is a constrained Prompt/Memory-level L4 prototype.** It implements harness search, an independent promotion gate, and version registration. The current source also provides optional experience retrieval and contrastive proposals. Sustained gains across generations, automatic code evolution, and joint model training each require their own experiments; saving memory or passing one gate does not establish them.

## 1. L1–L5 Classification

| Level | Core mechanism | Long-term self-evolution? |
| --- | --- | --- |
| L1 | Self-Refinement: regenerate after critic feedback | No |
| L2 | Test-Time Optimization: repeated search, comparison, and revision | Usually no |
| L3 | Persistent Memory / Skill Learning: retain experience for future tasks | Partly |
| L4 | Harness / Policy Evolution: modify the code, strategies, or workflows used by future tasks | Yes |
| L5 | Recursive Model–Harness Co-Evolution: continually update harnesses, rewards, training strategies, or models together | A stronger RSI objective |

This is an analytical framework, not an industry standard or a ranking of paper results. “Yes” at L4 describes the intended mechanism; actual long-term improvement still requires independent evidence. Higher levels do not necessarily implement every lower-level feature.

**Three kinds of memory differ:** StoryState holds facts and reference frames within one production; Harness Memory contains instructions that can persist across tasks; improvement experience stores Train provenance and evidence from parent/child candidates. StoryState alone does not establish L3, and the other two still need evidence of gains on new tasks.

## 2. Other L3/L4 Candidates Compared with H3-rsi

Levels below are this page's assessment. Different tasks and resource budgets prevent direct rankings across papers. Links identify papers or official implementations; suggested adaptations are proposals for H3-rsi, not completed integrations.

| Candidate / scope | Open-source status (2026-10-23) | Shared approach | Key difference | Potential adaptation for H3-rsi |
| --- | --- | --- | --- | --- |
| [CineForge](https://arxiv.org/abs/2608.29621); L4, video generation | **Unverified:** no official implementation located in this check; [paper](https://arxiv.org/abs/2608.29621) available | Use production evidence to revise reusable strategies and retain validated changes | Stage-specific CPPE patches and cross-story replay; H3-rsi candidates are currently restricted to six text components | Attribute recurring failures to production stages and validate changes across paired story evaluations |
| [VideoWeaver](https://arxiv.org/abs/2606.08091) / [code](https://github.com/JianhuiWei7/VideoWeaver); L3–L4, video generation | [Public implementation](https://github.com/JianhuiWei7/VideoWeaver); open-source license not identified in the inspected repository | Evaluate execution and video outputs to accumulate experience for future tasks | Learn composition/creator skills and dynamically orchestrate tools; H3-rsi has a fixed production order and natural-language Memory | Define explicit interfaces for tools and composition skills before allowing constrained workflow mutations |
| [SAGE](https://arxiv.org/abs/2608.17468); L3–L4, storyboarding knowledge | **Dataset only verified:** [PROSE](https://github.com/creDreams/PROSE), CC BY 4.0; full SAGE implementation not verified | Turn feedback into instructions or knowledge for later generation | Track rules adopted by narrative groups, update them from local feedback, and route by scenario; H3-rsi mainly tracks component diffs and parent/child candidates | Add rule IDs, adoption records, and local feedback instead of accumulating generic heuristics without attribution |
| [VideoHarness-RSI](https://arxiv.org/abs/2608.24302) / [code](https://github.com/Tencent/VideoHarness-RSI); L4, video understanding | **Yes:** [implementation](https://github.com/Tencent/VideoHarness-RSI), [Apache-2.0](https://github.com/Tencent/VideoHarness-RSI/blob/main/LICENSE.txt); third-party terms retained | Freeze the foundation model, search harnesses, and retain transferable changes | Search executable context-construction code; H3-rsi searches Prompt/Memory rather than Python | Start code-level experiments with typed reference-frame or observation-selection modules |
| [Video-RSI](https://arxiv.org/abs/2609.37950); L4, video understanding | **Yes:** [implementation](https://github.com/bingjunluo/Video-RSI), MIT | Use failure evidence to guide reusable strategy revisions | Actively revisit training videos to distinguish failure explanations and account for visual cost; H3-rsi probes still evaluate constrained text candidates through generation | Gather additional observations for failed segments to distinguish missing actions, sampling omissions, and judge errors |
| [ModularRSI](https://arxiv.org/abs/2609.14857) / [code](https://github.com/IQuestLab/ModularRSI); L4, general agents | **Public code, restricted:** research contributions [CC BY-NC 4.0](https://github.com/IQuestLab/ModularRSI/blob/main/LICENSE-MODULARRSI.md); Harbor-derived code Apache-2.0 | Contrast successes/failures, restrict modifications, and track experience across tasks | Evolve functional modules such as Agent Loop and Tool Use, then integrate them; H3-rsi modules are text fields, and displayed runs do not establish contrastive-mode gains | Aggregate evidence across tasks, intervene on one module, then assess integration conflicts and total probe cost |
| [HELIX](https://arxiv.org/abs/2608.13951) / [code](https://github.com/HKUDS/HELIX); general-agent L4, toward L5; **code repair, not video** | **Yes:** [implementation](https://github.com/HKUDS/HELIX), MIT | Preserve candidates, trajectories, and verification provenance | Explicit component/recipe interventions produce verifiable training records; one code-repair evolution round does not establish multigeneration joint training | Turn sibling successes, regressions, and near misses into traceable data, then separately validate model training and combined improvements |

**Task scope:** CineForge and VideoWeaver address video generation, SAGE addresses storyboarding, and VideoHarness-RSI/Video-RSI address video understanding. ModularRSI and HELIX address general coding/agent tasks; they are included only as transferable harness-evolution references, not as demonstrated video RSI systems.

**Status scope:** public source code, an open-source license, and complete experiment reproducibility are separate claims. “Unverified” does not mean confirmed closed-source; non-commercial-only code is distinguished from permissively licensed implementations.

[A²RD](https://arxiv.org/abs/2605.06924) and [CANVAS](https://arxiv.org/abs/2604.13452) also offer useful within-production multimodal memory and visual-continuity ideas. Cross-shot state management alone does not establish cross-task L3/L4. Likewise, accuracy gains in video understanding do not directly demonstrate video-generation gains.

## 3. H3-rsi: Implemented Capabilities and Evidence Boundaries

| Area | What the current implementation establishes | What it does not establish |
| --- | --- | --- |
| Candidate mutation | The [Codec](../../rsi-1st-impl-design/rsi_harness/optimization/codec.py#L77) deep-copies the base and replaces only five prompts and Memory; other configuration is inherited | Automatic modification of workflow code, routing, judges, reward weights, or model parameters |
| Five-segment production | The [Pipeline](../../rsi-1st-impl-design/rsi_harness/agents/pipeline.py#L61) and [StoryState](../../rsi-1st-impl-design/rsi_harness/longform/models.py#L185) generate outputs and update facts while keeping one harness fixed throughout the project | Production agents search or modify the harness during segments 2–5, or critic feedback necessarily triggers regeneration |
| Experience learning | [ExperienceAdapter](../../rsi-1st-impl-design/rsi_harness/optimization/experience.py#L124) provides optional snapshots, retrieval, Train-only contrastive probes, and export | These features were enabled in all `101214/101416/101704` runs or caused their recorded gains |
| Evaluation | [RewardBuilder](../../rsi-1st-impl-design/rsi_harness/reward/builder.py#L31) and [project aggregation](../../rsi-1st-impl-design/rsi_harness/longform/evaluator.py#L176) measure clip rewards and C/F/P/H separately | High boundary similarity proves correct action or complete storytelling; text-based story/editing scores equal human judgments of the finished movie |
| Version inheritance | [Baseline configuration](../../rsi-1st-impl-design/rsi_harness/longform/optimize.py#L124) supports `base_source: champion`, or a configured baseline with a separate store; resume preserves the original base | Sustained, unattended generalization gains across multiple generations have been demonstrated |
| Promotion | The [Gate](../../rsi-1st-impl-design/rsi_harness/longform/optimize.py#L191) and [store update](../../rsi-1st-impl-design/rsi_harness/longform/optimize.py#L860) are separate; registration rechecks the current champion against the run base | A winner automatically equals a champion, or gate acceptance guarantees a successful store update |

**Recorded experiments:** `101214` successfully registered `gepa-base.1`; `101416` winner `c1912a62` passed the gate, but a base/store mismatch prevented the champion update; `101704`'s Memory winner was rejected. See the [run index](subdocs_examples/README.md). These results support searchable, independently evaluated, versioned text harnesses; they do not independently demonstrate code-level L4 or L5.

## 4. Roadmap: Strengthen Evidence Before Expanding the Search Space

This is a proposed roadmap. Implementation, DGX verification, and research conclusions should be recorded separately.

| Stage | Next work | Controls and completion criteria |
| --- | --- | --- |
| P0: Reliability of existing mechanisms | Exercise champion inheritance and separate-store configurations on DGX; inspect baseline safeguards, resume, reflection failures, and budgets | A fixed base per run; no generation under a known mismatch; separately traceable gate verdicts and store-pointer outcomes |
| P1: Experience gains and generalization | Compare `off / experience / contrastive` under equal total budgets, using multiple seeds, new stories, and tasks from different sources | Account for probes, failed proposals, and cache behavior; report paired gains, regression rates, confidence intervals, human judgments, and cost rather than search scores alone |
| P2: Better diagnosis | Track rule adoption, stage-specific failure records, and action/narrative/audio checks; inspect rejected and high-scoring failure videos | Distinguish static presence from completed actions; report judge errors and skipped-payoff coverage; retain fixed independent evaluation |
| P3: Executable L4 | Initially allow mutation of one module: reference-asset selection, tool routing, or budget allocation | Typed interfaces, scope restrictions, isolated execution, replay, regression gates, and rollback; demonstrate gains on unseen tasks under matched budgets |
| P4: Multigeneration improvement | Use each actually promoted champion as the next run's base; achieve at least two consecutive successful promotions | Complete lineage, fresh Promotion/Test tasks per generation, and cost reports; rejected candidates and manual combinations do not count as successful automatic generations |
| P5: L5 research | Build training data from verified trajectories, specify whether to train the agent policy or H3 weights, and integrate verl-omni | Actual checkpoints and data provenance; four-combination ablation followed by an independent gate for the complete system |

The L5 design follows [page 15](15-joint-optimization.md): `(H0, θ0)`, `(H1, θ0)`, `(H0, θ1)`, and `(H1, θ1)` separate harness effects, model effects, and their interaction. Reliable trajectories and evaluation should precede adaptations of [VideoGen-Agent](https://arxiv.org/abs/2609.24997)'s tool-policy learning or [MVAgent](https://arxiv.org/abs/2609.30609)'s shot-level credit assignment. Connecting a training interface alone does not demonstrate L5.

## 5. Suggested Project Positioning

**“Evidence-grounded video-generation harness evolution with independent evaluation and version management: a Prompt/Memory-level L4 prototype.”**

Current strengths are an auditable experimental chain and a constrained optimization boundary. The main gaps are evaluation reliability, multigeneration/cross-domain evidence, executable-module evolution, and model training. Future work should address these gaps through falsifiable controlled experiments.

---

[Contents](README.md) · [Previous](17-findings-and-pilot-prompts.md) · [Full research analysis](../rfc748/H3_RSI_2026_Research_and_Conversation_Full.md)
