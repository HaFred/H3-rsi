# RSI × MiniMax-H3: Implementation Introduction

This directory presents the supplied presentation as an English introduction to the H3-rsi implementation. Each of the original 17 pages has its own Markdown file, in the presentation order; page 18 is a later research analysis supplement. The accidental duplicate formerly at page 4 was removed from the source before this export.

Read pages 1–7 for the definition, architecture, search boundary, and evaluation protocol; pages 8–10 for short-clip evidence; pages 11–14 for long-form execution and evidence; pages 15–17 for the proposed joint-optimization design, limitations, and pilot prompts; and the supplemental page 18 for RSI levels, related work and the research roadmap.

## Contents

| PPTX page | English Markdown |
|---|---|
| 01 | [RSI × MiniMax-H3: Self-Improving Video Generation](01-overview.md) |
| 02 | [RSI and AI Agents: The Project Definition](02-rsi-definition.md) |
| 03 | [Harness RSI: Generation, Evaluation, and Promotion](03-harness-workflow.md) |
| 04 | [Inference (Agent Tool) and Evaluation Services](04-service-topology.md) |
| 05 | [The Harness Search Boundary](05-search-boundary.md) |
| 06 | [GEPA: Search, Evaluation, and Promotion Loops](06-gepa-loops.md) |
| 07 | [Four Data Splits and the S / R Scores](07-data-splits-and-scores.md) |
| 08 | [Harness Improvements: Short-Clip Comparison](08-short-clip-results.md) |
| 09 | [Short-Clip Videos from Three Harness Versions](09-short-clip-videos.md) |
| 10 | [How Harness Changes Propagate Downstream](10-prompt-propagation.md) |
| 11 | [Long-Form Production: State Verification and First-Frame Handoff](11-long-form-production.md) |
| 12 | [Long-Form Example: The town-fair Rewrite Lineage](12-town-fair-lineage.md) |
| 13 | [town-fair Videos Across the Recursive Lineage](13-town-fair-videos.md) |
| 14 | [Long-Form Optimization: GEPA and Project Execution](14-long-form-optimization.md) |
| 15 | [Harness and Model RSI: Parallel Optimization](15-joint-optimization.md) |
| 16 | [Experimental Boundaries and Implementation Roadmap](16-evidence-and-roadmap.md) |
| 17 | [Initial Findings, Data Limitations, and Pilot Prompts](17-findings-and-pilot-prompts.md) |
| 18 (supplement) | [Video RSI Levels, L3/L4 Comparisons, and the H3-rsi Roadmap](18-rsi-levels-comparison-and-roadmap.md) |

## Worked examples

Start with the [run and harness index](subdocs_examples/README.md) for candidate identities, inheritance, durations and promotion outcomes.

| Related pages | Detailed example |
| --- | --- |
| 04, 07-10: agent inputs, clip scores, videos and prompt propagation | [Single-clip script rewrite: prompts, agent outputs, Base/rewrite H3 payloads and videos](subdocs_examples/08-script-rewrite-prompt-lifecycle.md) |
| 04, 07, 11-14: segment lifecycle, project scores, state, handoff and stitching | [Five-clip town-fair production: Base vs Script rewrite](subdocs_examples/11-town-fair-longform-production.md) |
| 07, 11-14: held-out regression, memory changes and promotion safeguards | [Five-clip tea-farm production: Base vs memory rewrite](subdocs_examples/11-tea-farm-longform-production.md) |

The town-fair example includes source snapshots, segment previews and the recovered DGX traces, H3 requests and project scores for both arms.

## Reading the evidence

- Harness search is demonstrated with frozen H3 weights. Model-weight optimization and the four-combination gate are a proposed next stage.
- Use R_clip/S_clip for a single clip and R_project/S_project for a complete project. S_clip derives from the same search case's R_clip and axis penalty. Both project scores combine current project metrics and share C = mean(R_clip). Independent Promotion measures the appropriate reward on held-out cases; search never uses those future results. See [page 7](07-data-splits-and-scores.md).
- The recovered `101416` run completed Promotion and Test after interruption and resume. Its final winner passed the gate, but the champion store stayed unchanged because its current champion did not match the expected base. The original presentation predates that completed outcome; see the [run summary](subdocs_examples/README.md#how-search-and-promotion-relate).
- Long-form lineage S_project/R_project values are averages over four Validation projects, rather than scores for the displayed town-fair videos.
- Repeated champion inheritance and cumulative gains on new works remain unverified. Test reports generalization evidence; it does not establish global optimality.

## Source and conversion

Source: `RSI_MiniMax_H3_RFC748_zh_white_v17_updated.pptx`, as supplied locally after duplicate-page removal. The PowerPoint file is not copied into this repository.

Text is translated and organized into prose, tables, and workflow steps. Page 7's chart numbers are transcribed from its native chart data. Embedded images and MP4s were extracted without modification. Longform presentation media remains in `assets/`; byte-identical short-clip videos use the existing evaluation files linked on page 9. Images remain in their original visual form, and local video playback depends on the viewer. Preview links supplied by the presentation are preserved without asserting their current remote availability. Video and preview ordering follows presentation drawing/relationship order; the visible candidate labels and comparison tables provide context.

Source speaker notes were used to retain important interpretation limits and clarify terminology; historical/repeated editing notes are not reproduced as narrative. Decorative diagram legends are represented through the component/role descriptions rather than repeated on every page.

Page 9 includes a subsequent clarification of the video-axis metric: its three frame-level scoring dimensions, normalization, evaluation limits, and relationship to the composite R_clip score. This clarification is also present in the revised presentation.
