# 04. Inference (Agent Tool) and Evaluation Services

Source: PPTX page 4 of 17.

| Role | Model / responsibility | Endpoint | GPU |
|---|---|---|---|
| Video generation | MiniMax-H3 fl2va, SGLang TP2 | `:30010` | a/b |
| Production text | neohorse-1-4B | `:8000` | c |
| Reflection | Qwen3.5-9B; propose one-component Harness rewrites | `:8001` | c |
| Decisions | NeoHorse-Jev-4B; story / gates | `:8085` | c |
| Visual judge | UnifiedReward-2.0-qwen3vl-2B; video axis | `:8090` | c |

One production text model (`neohorse-1-4B`, `:8000`) serves five workflow agents. Each call has two inputs: an agent-specific **system message** from the selected harness's `prompts.<agent>` plus shared `memory`, and a **user message** assembled from case requirements or preceding agent outputs. The first three agents produce H3's generation description; critic and editor run after generation and provide text diagnostics.

| Agent | System prompt path in the selected harness | User-message inputs and their origin | Output / destination | Input assembly code | Does this output serve as a prompt in H3 inference? |
| --- | --- | --- | --- | --- | --- |
| Scriptwriter (`script`) | `prompts.script` + `memory` | `Idea: <task.prompt>` plus target duration and `planning.max_shots`. For single clips, `task.prompt` is the JSONL row's `prompt`; for five-clip projects, it is the composed segment brief described below. | Script -> storyboard and director | `rsi_harness/agents/script.py`, `ScriptAgent.run` | **No.** The script is an intermediate planning document consumed by later text agents; H3 has no separate script-role prompt input in this workflow. |
| Storyboard planner (`storyboard`) | `prompts.storyboard` + `memory` | Generated script + target total duration. The original case fields reach this agent through the script. | JSON shot list -> director and critic | `rsi_harness/agents/storyboard.py`, `StoryboardAgent.run` | **No.** The JSON shot list is consumed by the director; shot/timing details may be incorporated into the final H3 prompt, but this JSON is not submitted as a separate storyboard-role prompt. |
| Director (`director`) | `prompts.director` + `memory` | Generated script + storyboard JSON + target duration, aspect ratio and selected generation mode. | Final generation text -> H3 request's `prompt` | `rsi_harness/agents/director.py`, `DirectorAgent.run` | **Yes.** The director's enhanced text is submitted directly as H3's `prompt`. Its three requested fields match MiniMax's documented base-mode prompt structure; mode-specific formatting and reference binding are not enforced by a dedicated compiler/validator on the current harness path. |
| Text critic (`critic`) | `prompts.critic` + `memory` | Generated script + storyboard JSON + director's generation description. The implementation does not read video/frames from its `artifact` argument. | Defect report -> Jev diagnostic gates and editor | `rsi_harness/agents/critic.py`, `CriticAgent.run` | **No.** The defect report goes to diagnostic gates and the editor after generation; it is not submitted to H3 as a critic-role prompt or a revision prompt in this workflow. |
| Editing adviser (`editor`) | `prompts.editor` + `memory` | Critic report + a fixed instruction to pick the best candidate and advise on shot lengths. | Text editing advice; does not modify the generated clip or timeline | `rsi_harness/agents/editor.py`, `EditorAgent.run` | **No.** The editing advice is recorded after generation; it is not submitted to H3 as an editor-role prompt and does not trigger another H3 inference. |

**What H3 consumes.** The rightmost column compares the agents' **output texts** with H3 inference inputs. Only the director's output is submitted directly as the generation `prompt`; script and storyboard outputs inform it through the text pipeline. These are not five separate H3 role/system prompts. The official [H3-Base input specification](https://huggingface.co/MiniMaxAI/MiniMax-H3#local-deployment-of-h3-base) accepts text with optional media conditions; hosted [H3-Context-IR](https://huggingface.co/MiniMaxAI/MiniMax-H3#h3-context-ir) prepares an H3-ready representation. The current director targets MiniMax's documented three-field base-mode format: `integrated_multimodal_description`, `overall_soundscape`, and `non_diegetic_music`. Following this public format is distinct from reproducing the behavior or quality of MiniMax's closed Context-IR compiler. The harness's `prompts.*` system messages instruct the production text agents and are not themselves passed to H3.

**Prompt format depends on generation mode.** MiniMax's [base-mode guide](https://huggingface.co/MiniMaxAI/MiniMax-H3/blob/main/docs/VIDEO_PROMPT_WRITING_GUIDE_base_en.md) specifies the three fields above for T2VA, with a mode-specific image-alignment instruction added for first/last-frame generation. The [full-reference guide](https://huggingface.co/MiniMaxAI/MiniMax-H3/blob/main/docs/VIDEO_PROMPT_WRITING_GUIDE_ref_en.md) specifies six sections for Ref2VA: `subject_definitions`, `summary`, `retention_analysis`, `detailed_description`, `overall_soundscape`, and `non_diegetic_music`. The six-section [open Context-IR example in RFC #500](https://github.com/verl-project/verl-omni/issues/500#issuecomment-5506568087) follows that full-reference structure; it does not imply that every H3 generation mode needs six sections. Our longform backend uses the `fl2va` route with one first-frame image, which corresponds to I2VA prompt semantics.

**Proposed compiler boundary (not implemented in the recorded runs).** Keep the agents' creative planning, then compile their generation decisions into H3's documented mode-specific prompt format before inference. A pinned open Context-IR compiler would make formatting, reference labels and attachment binding explicit and validated. The intended path is **generation decisions -> mode-aware compilation and validation -> rendered prompt + matching media conditions -> H3**. The current path forwards the director's text directly. Alignment with the public prompt format does not establish equivalence to the closed compiler; improvements from adding this boundary require a separate evaluation.

The prompt fields above live in the selected harness YAML: compare the single-clip [base](../base.yaml) and [search winner](../best.yaml), or the five-clip [base](../longform/101704_1strun/base.yaml) and [search winner](../longform/101704_1strun/best.yaml). `AgentContext.system_prompt` in `rsi_harness/agents/base.py` appends memory to each agent's system message. Implementation paths refer to the sibling `rsi-1st-impl-design` checkout; case inputs and harness snapshots are published here. Both runs search `script`, `storyboard`, `director` and `memory`; critic/editor prompt fields are fixed.

**Where the case inputs enter.** The [single-clip field reference](../benchmarks/README.md#single-clip-field-reference) defines the JSONL task inputs; only its `prompt` supplies the Script agent's idea text. IDs and split/source fields support provenance and evaluation control, while `task_type` and `extra.auto_route` control generation mode. The [ProjectSpec field reference](../benchmarks/README.md#projectspec-field-reference) defines the five-clip YAML inputs. For each segment, `compose_brief` in `rsi_harness/longform/producer.py` combines project title and segment position/ID, `continuity_contract.project_world`, already confirmed StoryState facts, `segments[].purpose` and `prompt`, any prior-frame handoff instruction, and nonempty `audio_policy` / `visual_policy`. That assembled brief becomes `task.prompt` in the Script agent's user message.

`recurring_motifs` is metadata and is not separately injected by the current brief composer. Evidence/payoff questions go to the visual judge after generation; their question text is not automatically inserted into these agents' user messages. Evidence-confirmed fact values can enter subsequent Script briefs through StoryState. See the [question-flow table](../README.md#23-five-clip-visual-evidence-and-its-video-mapping) for their role in evaluation and reflection.

The resulting path is **case requirement / segment brief -> script -> storyboard -> director prompt -> H3**. H3 receives the director's derived text plus generation parameters and, for later longform segments, a prior-frame condition. It does not receive these YAML fields as separate system messages; preservation of their instructions depends on the text agents' outputs.

The short-clip run used **65 H3 requests**: 45 for search, 12 for promotion, and 8 for Test, against a total limit of 80 (60 + 12 + 8).

- Search includes iteration 0 Base evaluation and iterations 1–13. Each attempted rewrite compares parent/child on the same two Train tasks; passing candidates receive full evaluation on two Validation tasks. Cache hits do not add physical H3 requests.
- Four new candidates reached full validation. The winning lineage changed script at iteration 1 and director at iteration 10.
- Promotion: 6 tasks × 2 versions × 1 seed = 12 requests.
- Test: 4 tasks × 2 versions × 1 seed = 8 requests.

The two versions are Base and winner. Promotion/Test do not rewrite the Harness; eight Test requests do not mean eight search rounds. Each H3 request produces one clip: 124 frames, about 5.17 seconds, 1344 × 768, 24 fps, and 10 inference steps.

---

[Contents](README.md) · [Previous](03-harness-workflow.md) · [Next](05-search-boundary.md)
